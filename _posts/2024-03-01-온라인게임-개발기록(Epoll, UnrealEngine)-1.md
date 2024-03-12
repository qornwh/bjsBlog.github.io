---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-1
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal, server, c++]
---

# Epoll을 이용한 소켓 통신

## 서버에서의 소켓 설정

온라인 게임서버에서의 통신은 실시간으로 이뤄져야 된다. 실시간으로 통신할려고 하면 Socket을 non-block으로 설정하고, epoll의 수신데이터가 버퍼에 쌓이면 한번만 감지 해서 알려주면 되기 때문에 '에지 트리거' 모드로 설정할 필요가 있다.

```cpp
bool SetNonBlockSocket(int _serverFd)
{
  int sFlags = fcntl(_serverFd, F_GETFL);
  sFlags |= O_NONBLOCK;
  if (fcntl(_serverFd, F_SETFL, sFlags) < 0)
    return false;
  return true;
}
```

리눅스에서 소켓은 FileDescriptor로 작동하므로, 현재 fd 상태를 들고온뒤 non-blocking설정 성공여부를 판단하는 함수이다.

```cpp
socket(PF_INET, SOCK_STREAM, 0); // ip4, tcp 사용하는 소켓 생성코드이다.
setsockopt(_serverFd, SOL_SOCKET, SO_REUSEADDR, &option, sizeof(option)) // 프로그램 정상 종료시에도 소켓은 잠시동안 물고 있기때문에 재할당 가능 소켓으로 변경하는 옵션이다(SO_REUSEADDR)
setsockopt(_serverFd, SOL_SOCKET, SO_REUSEPORT, &option, sizeof(option)) // 포트 재할당이 가능하게 하는 옵션이다.(SO_REUSEPORT)
```

```cpp
bool EpollFD::Register(int socketFd)
{
  /* --- 일단 클라 서버 통일 ----*/
  struct epoll_event events;
  events.events = EPOLLIN | EPOLLRDHUP | EPOLLET; // 수신데이터 | 연결 끊김 감지 | 에지 트리거
  events.data.fd = socketFd;

  if (epoll_ctl(_epollFd, EPOLL_CTL_ADD, socketFd, &events) < 0)
  {
    close(socketFd);
    return false;
  }
  return true;
}
```

위 함수는 소켓에 epoll 모드로 설정하는 코드이다. epoll_event로 \_epollFd에 설정후에 epoll_ctl 함수로 소켓을 등록하는 코드이다.

## 서버 클라이언트 통신

**서버의 전체적인 통신 과정이다.**

1. ThreadManager로 Service 생성
2. Service에서 연결시 Session(클라이언트) 생성, 읽기시 Session(클라이언트)에서 읽기 시작
3. 패킷을 header로 분해 id, size
4. 각 패킷 id에확인후 Room으로 전달.
5. Room에서는 BroadCast로 각 Session(클라이언트)에 전달및 몬스터 이동 코드 전달.
6. Service, Session, Job등 동적할당으로 객체를 생성은 std::shared_ptr로 작성됨.

### 서버 소켓 감지

서버에서는 소켓을 epoll_wait함수로 connect, read, disconnect를 감지하게된다. epoll_wait함수로 select모델과 달리 감지가 일어나게된 소켓들만 fd를 가져올수 있게된다.

```cpp
epoll_wait(_epollFd, _epollEvents, MAX_CLIENT, TIMEOUT); // _epollEvents 2번째 파라미터에 감지된 모든 소켓이 리스트가 담겨진다.
```

현재 서버에서는 <[Dispatch()](https://github.com/qornwh/BSGameServer/blob/6cc18c87e9192adb951f1a2c0836b0ee7ca4180d/CoreLib/Service.cpp#L86)> 함수를 이용해서 매시간 epoll_wait을 호출하고 있다.

### lock 구현

read write lock을 구현하기위해 원자성으로 작동되는 compare_exchange_strong함수를 사용했다. 방금 atomic변수에서 가져온 현재 atomic변수값이 같은지확인후 같으면 예측되는 값으로 변경하고 아니면 lock 실패로 스핀락으로 구현했다. 여기서 나오는 `원자성`이라는 말은 visualStudio에서 어셈블리어로 보면 대입(=)과 같은 코드는 어셈블리 명령이 1개지만, 덧셈, 뻴셈등은 어셈블리로 보았을때 명령이 여러개로 나타나므로, 멀티스레드 contextSwitch에서 공유자원이 안전하게 동작되지 않는다.

ReadLock, WriteLock 구현은 readLock을 잡은경우에는 read-read-read로 읽기만 하는 경우는 계속 잡을수 있도록 구현했고, writeLock을 잡은경우에는 읽기만 가능하고 모든 readLock이 풀려야 writeLock을 풀도록 만들었다.

- read -> wirte (x)
- write -> read (o)

![lock](/assets/img/lock.png)

ReadLockGuard, WriteLockGuard을 구현하는데 생성자가 소멸될때 락이 해지되도록 RAII패턴으로 구현했다.

```cpp
class ReadLockGuard
{
public:
	ReadLockGuard(Lock& lock, const char* name) : _lock(lock), _name(name) { _lock.ReadLock(); }
	~ReadLockGuard() { _lock.ReadUnLock(); }
private:
	Lock& _lock;
	const char* _name;
};

// 사용
{
  ReadLockGuard read(lock, "read"); // 락 잡음 여기서는 안전함.
}
// 코드 범위를 벗어나면 ReadLockGuard 생성자가 소멸될때 락도 해제됨!!
```

`마지막으로 lock을 CAS로 스핀락을 구현했지만, 완벽하게 차단되는지는 제대로된 테스트가 필요할거 같다.`

### JobQueue 구성

JobQueue은 멀티스레드 환경에서 contextSwitch되면 공유자원을 침범하는 하게 된다. 이를 방어하기 위해서. JobQueue는 push, execute될때 Lock을 사용해 생성자소비자 패턴처럼 작성되어 있다. 나중에 나올 Room(JobQueue상속)에서 사용된다.

![JobQueue](/assets/img/JobQueue.png)

```cpp
using Func = std::function<void()>;

class Job
{
public:
  // no params
  template <typename T>
  Job(void (T::*funcPtr)(), std::shared_ptr<T> instance)
  {
    _func = [funcPtr, instance]
    {
      (instance.get()->*funcPtr)();
    };
  }

  template <typename T, typename... Args>
  Job(void (T::*funcPtr)(Args...), std::shared_ptr<T> instance, Args... args)
  {
    _func = [funcPtr, instance, args...]
    {
      (instance.get()->*funcPtr)(args...);
    };
  }

  void Execute()
  {
    _func();
  }

private:
  Func _func;
};
```

위 클래스가 Job이 구현된 코드이다. 첫번째 Job생성자는 function의 파라미터가 없고 해당되는 shared_ptr을 파라미터로 가지고, 두번째 Job 생성자는 function의 파라미터가 추가된 모습이다.

[**네트워크 라이브러리 코드 CoreLib폴더 참조**](https://github.com/qornwh/BSGameServer/tree/main/CoreLib)
