---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-2
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal]
---

# 게임 패킷 구현 및 멀티 스레드

## 패킷 구성

패킷은 패킷 헤더 + 패킷 내용으로 구성된다. 패킷 헤더에는 id(2byte) + 패킷 크기(2byte)로 구성되고 패킷 내용에는 이동, 채팅, 공격 메시지에 대한 내용이 담겨있다.
![Packet](/assets/img/Packet.png)

## 패킷 Read

![ReadBuffer](/assets/img/ReadBuffer.png)
먼저 Tcp로 통신하게 되므로 패킷이 오는 순서는 보장되어 있지만, 중간이 나눠져서 올 수 있다. 예를들면 다음과 같다. ex) 클라 50보냄 => 서버 25받음, 서버 25받음.이런식으로 데이터를 받을경우 buffer에 데이터를 쌓아둔뒤에 패킷 크기만큼 읽어서 처리한다.

## 패킷 Send

![SendBuffer](/assets/img/SendBuffer.png)
SendBuffer는 항상 메모리를 할당해서 버퍼를 만드는게 아닌 풀링을 해서 구현했다. 위 그림처럼 SendBufferManager는 전역에 하나 만들어 두고, vector로 SendBufferChunk가 있다. SendBufferChunk는 SendBuffer가 패킷크기마다 다르기때문에 일정범위의 1byte Array를 만들고 패킷 사이즈만큼 Senbuffer를 만들고 한번 전부다 라이팅되면 다시 전역에 push하는 방법으로 구현했다.

## 멀티 스레드 구현

멀티스레드는 구현은 스레드와 스레드에서만 전역으로 사용되는 thread_local을 사용하였다. thread_local로 선언한 전역 변수는 id와 멀티스레드에서 sendbuffer에서 lock을 걸지 않도록 SendBufferChunk을 두었다.
먼저 전역에 스레드를 생성할수 있는 threadManager 클래스를 선언하고 threadManager에서 스레드를 생성하는 방식으로 구현했다. 생성할때 id 초기화와 callback으로 넘어온 함수를 실행하고 종료될때 동적 할당된 thread_local이 있으면 할당 해제하도록 구현했다.

```cpp
// callback 스레드에서 처리할 함수
  void CreateThread(function<void()> callback)
  {
    threadId.fetch_add(1);

    threadList.push_back(thread([=]() {
      ThreadTLS();
      callback();
      ThreadDestory();
    }));
  }
```

## 프로그램 CRASH

서비스가 실행이 안된다거나, socket 초기화, 멀티스레드에서 공유자원 획득에 맞지 않는 현상이 일어날때 사용하기 위해 구현했다.
프로그램을 바로 종료가 필요한경우 `ASSERT_CRASH` 함수로 종료한다.

```cpp
#define CRASH(cause)
{
  volatile int* crash = nullptr;
  *crash = 0xDEADBEEF;
  exit(1);
}

#define ASSERT_CRASH(expr)
{
  if (!(expr))
  {
    CRASH("ASSERT_CRASH");
  }
}
```

[**네트워크 라이브러리 코드 CoreLib폴더 참조**](https://github.com/qornwh/BSGameServer/tree/main/CoreLib)
