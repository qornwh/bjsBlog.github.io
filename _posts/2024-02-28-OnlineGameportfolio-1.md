---
title: 온라인게임 포트폴리오-1
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, boost, Unity, googleProtobuf, server, c++]
---

# 네트워크 라이브러리 구현및 중요기능(CoreLib)

---

## 네트워크 라이브러리 목차

---

- 서버 Serivce, 클라이언트 Session 구현(boost::asio)
- read, write buffer를 구현
- thread, type 지정

### 서버 Serivce, 클라이언트 Session 구현(boost::asio)

---

먼저 boost::asio를 이용하고 tcp에 non-blocking으로 서버를 구현.
`Service`에서 클라이언트 accept, packet(message) broadcast, 클라이언트 disconnect등을 담당하고, `Session`에서는 서버로부터 packet(message) read, send 처리를 담당한다.

아래 그림처럼 구성되어 있다.
![SeviceSession](/assets/img/online2/SeviceSession.png)
_SeviceSession_
먼저 `Service`에서 `Session`을 생성하고 boost::asio의 `acceptor.async_accept`으로 먼저 `accept`를 등록한다. 그다음 클라이언트와 `Connect`이 read write가 가능하고. unity에서 packet을 send하면 `Session`에서 패킷을 분석한뒤에 `BroadCast`모든 `Session`에 필요한 패킷을 전달하게 된다.

마지막으로 `lock`은 `mutex`를 사용해서 구현. `CAS`로 스핀락이나 `InterlockedAdd`으로 구현할수 있지만, 사실 이부분은 확신이 없어서 그냥 기본 `mutex`를 이용해서 구현. `lock` 사용은 `ReadLockGard`, `WriteLockGard`로 사용되는데 c++에서 존재한 [`RAII`](https://en.cppreference.com/w/cpp/language/raii)패턴을 이용해서 구현.

### recv, send buffer를 구현

---

`RecvBuffer`는 Tcp프로토콜이 데이터가 한번에 들어오는게 아니라 쪼개저서 들어올수 있으므로 사용했다. Packet은 나중에 나오지만 Packet을 구성하는데 제일 앞에 packetId, packetSize순서대로 들어오기 때문에, 데이터가 들어올때 packetSize를 체크해서 size만큼 들어오지 않으면 ReadBuffer에 킵 해두었다가 다시 데이터가 들어와서 ReadBuffer를 비우는 형식으로 구현.  
`RecvBuffer`의 처리는 `OnWrite` 함수로 들어온 데이터 만큼 읽고 `OnRead`에서 패킷 사이즈만큼 있으면 처리한다. 그 다음 `OnClean`함수로 buffer를 ReadSize, WriteSize가 같으면 초기또는 일정 범위 동안 쌓여 있으면 ReadSize를 0으로 바꾸고, WriteSize 를 `WriteSize - ReadSize`로 바꾸고 메모리를 앞으로 당긴다.

![ReadBuffer](/assets/img/online2/ReadBuffer.png)
_ReadBuffer_

`SendBuffer`는 3가지의 클래스로 관리된다. `SendBuffer`클래스는 버퍼가 얼마만큼 할당받고 쓰여지는지 이용되고, `SendBufferChunk`클래스는 `SendBuffer`를 할당해주고 실제로 버퍼를(4096) 들고 있다. 마지막으로 `SendBufferChunk`를 관리하는 `SendBufferManager`가 `TSL`가 관리하는 `SendBufferChunk`를 리스트로 들고 있어서 `SendBufferChunk`를 할당해주는 역할이고, `SendBufferChunk`가 가르키는 포인터가 사라질때 다시 `TSL`가 관리하는 `SendBufferChunk`에 push되는 방식이다.  
정책은 다음과 같다.

1. 초기 프로그램 실행될때 미리 `SendBufferManager`는 `SendBufferChunk` 일정 개수만큼 생성한다.
2. `SendBufferChunk`는 풀링으로 관리되면 할당될때 `TLS_SendbufferChunk`에 남은게 있으면 pop으로 가져오고, 없으면 동적할당된다. `SendBufferChunk`가 사용되고 가르키는 포인터가 없으면 할당해제되는게 아니라 다시 `TLS_SendbufferChunk`에 들어가게 된다.

`SendBufferChunk`를 할당받는 코드는 다음과 같다.

```cpp
SendBufferRef SendBufferManager::Open(uint32 size)
{
  SendBufferChunkRef chunk;
  {
    if (_chunks.empty())
    {
      chunk = _chunks.emplace_back();
      _chunks.pop_back();
    }
    else
      chunk = CreateChunk();
  }

  if (chunk)
  {
    chunk->Reset();
    return chunk->Open(size);
  }

  return nullptr;
}

// 할당해제시 ReleaseBuffer가 발생
void SendBufferManager::ReleaseBuffer(SendBufferChunk* chunk)
{
  SendBufferChunkRef sendBufferChunk = SendBufferChunkRef(chunk, ReleaseBuffer);
  TLS_SendBufferManager->_chunks.push_back(sendBufferChunk);
}

// SendBufferChunkRef == boost::shared_ptr<class SendBufferChunk>, new로 생성, 해제는 ReleaseBuffer로 다시 tls로 들어감
SendBufferChunkRef SendBufferManager::CreateChunk() const
{
  return SendBufferChunkRef(new SendBufferChunk(), ReleaseBuffer);
}
```

### thread, type 지정

---

`Thread`는 `ThreadManager`에서 생성한다. TLS로 thread id, Sendbuffer를 풀링하기 위한 변수 2개가 있고. `boost::asio::thread_pool`을 사용했다. 아래는 스레드 생성시 사용되는 함수이다.

```cpp
// 스레드 생성
void ThreadManager::CreateThread(function<void()> callback)
{
    boost::asio::post(_threadPool, [=]()
    {
        ThreadTLS(); // thread id, Sendbuffer 초기화
        callback(); // 스레드에서 실행할 콜백
        ThreadDestory(); // Sendbuffer 동적할당 해제
    });
}

// 프로그램 종료시
void ThreadManager::ThreadJoinAll()
{
    _threadPool.join();
}

boost::asio::thread_pool _threadPool;
```

`Type`은 `Types.h`에 선언되어 있다. boost의 자료형을 사용했고, `atomic`만 template으로 모든 타입을 받을수 있게 구현. 스마트 포인터는 모두 boost::shared_ptr, boost::weak_ptr을 사용했다.

```cpp
using BYTE = boost::uint8_t;
using int8 = boost::int8_t;
using int16 = boost::int16_t;
using int32 = boost::int32_t;
using int64 = boost::int64_t;
using uint8 = boost::uint8_t;
using uint16 = boost::uint16_t;
using uint32 = boost::uint32_t;
using uint64 = boost::uint64_t;

template <typename T>
using Atomic = std::atomic<T>;

using SessionRef = boost::shared_ptr<class Session>;
```

[**CoreLib 코드 참조**](https://github.com/qornwh/GameServerProject/tree/main/CoreLib)

### 구현 및 기능

---

1. [**프로젝트 소개**](/bjsBlog.github.io/posts/OnlineGameportfolio-0)
2. [**네트워크 라이브러리 구현및 중요기능(CoreLib)**](/bjsBlog.github.io/posts/OnlineGameportfolio-1)
3. [**게임서버 구현및 중요기능(GameServer)**](/bjsBlog.github.io/posts/OnlineGameportfolio-2)
4. [**피격판정 기능(GameEngine)**](/bjsBlog.github.io/posts/OnlineGameportfolio-3)
5. [**클라이언트 통신(Unity)**](/bjsBlog.github.io/posts/OnlineGameportfolio-4)
