---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-4
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal]
---

# 언리얼 엔진에서의 소켓 통신및 플레이 영상

## 언리얼 엔진에서의 소켓 통신

### 클라이언트 구조

클라이언트 동작은 간단하게 구성했다.

1. 게임선택창 원하는 캐릭터 선택
2. 서버와 클라이언트 소켓 Connect및 몬스터, 다른 유저 load
3. 게임맵으로 변경
4. 클라이언트 행동
   1. 이동
   2. 채팅
   3. 공격

### 클라이언트 소켓 연결

언리얼엔진에서 제공되는 `FSocket`을 이용하였고 논블로킹, 동기 방식으로 구현했다. 사실 동기 방식이지만 `FSocket.Wait` 함수로 읽어야할 데이터가 있으면 `Recv`함수를 호출하기는 한다. 연결 코드는 다음과 같다(`_clientSocket`변수가 `FSocket`이다).

```cpp
void UMyGameInstance::ConnectClientSocket()
{
  isConnect = false;
  const int32 port = 12127;
  const FString Address = TEXT("172.30.1.53"); // 리눅스 서버 주소
  FIPv4Address Ip;
  FIPv4Address::Parse(Address, Ip); // 서버의 IP 주소 입력

  TSharedRef<FInternetAddr> ServerAddress = ISocketSubsystem::Get(PLATFORM_SOCKETSUBSYSTEM)->CreateInternetAddr();
  ServerAddress->SetIp(Ip.Value);
  ServerAddress->SetPort(port); // 서버 포트 번호 설정

  _clientSocket = FTcpSocketBuilder(TEXT("MyFSocketActor")).AsNonBlocking().AsReusable();
  _clientSocket->Connect(*ServerAddress);
  // ...
}
```

### 클라이언트 소켓 Recv

클라이언트의 소켓에 관한 모든 함수는 `MyFSocketAcotr`에 있다. 이 액터를 맵에 넣어서 tick이 돌때마다 읽기를 수행하는 방식으로 구현했다.
![FSocket실행](/images/FSocket실행.png){: width="400" height="250"}

`FSocket`의 `Wait`함수로 `ESocketWaitConditions::WaitForRead` 매개변수로두고 1초동안 읽어야할 데이터가 있으면 Recv함수가 호출되고, 서버와 마찬가지로 buffer에 읽을 byte배열을 두고 `FPacketHeader`, `ReadMessageHandler` 패킷헤더와 패킷을 분석하는 방식으로 구현했다.

```cpp
void AMyFSocketActor::Tick(float DeltaTime)
{
  Super::Tick(DeltaTime);

  if (IsConnect)
  {
    int byteRead = 0;
    if (_clientSocket->Wait(ESocketWaitConditions::WaitForRead, FTimespan(1)))
    {
      if (_clientSocket->Recv(reinterpret_cast<uint8*>(__readBuffer), 4096, byteRead))
      {
        if (byteRead > 0)
        {
          int useLen = 0;
          while (useLen < byteRead)
          {
            int dataSize = byteRead - useLen;

            if (dataSize < sizeof(FPacketHeader))
              break;

            FPacketHeader* _header = reinterpret_cast<FPacketHeader*>(&__readBuffer[useLen]);
            if (dataSize < _header->size)
            {
              // 일부분만 데이터 들어오는경우 처리해야됨
              UE_LOG(LogTemp, Warning, TEXT("sdfsdfsdf !!!! %d"), useLen);
              break;
            }

            // 패킷 메시지 읽기
            ReadMessageHandler(reinterpret_cast<uint8*>(&__readBuffer[useLen]), useLen);
            useLen += _header->size;
          }
        }
      }
    }
  }
}
```

패킷을 읽어주는 함수로 `ReadBufferPtr`와 `ReadBufferStr16`문자열(이름, 채팅)을 도와주는 함수를 구현했다. char이 아닌 tchar인 이유는 언리얼에서 유니코드(UTF-16)으로 되어 있어서 tchar을 선택했다.

```cpp
template <typename T>
T AMyFSocketActor::ReadBufferPtr(uint8* buffer, int& offset)
{
  int _ptr = offset;
  offset += sizeof(T);
  T* temp = reinterpret_cast<T*>(&buffer[_ptr]);
  return static_cast<T>(*temp);
}

FString AMyFSocketActor::ReadBufferStr16(uint8* buffer, int& ptr, int32 len)
{
  FString ChatString;
  ChatString.AppendChars(reinterpret_cast<const TCHAR*>(&buffer[ptr]), len / sizeof(TCHAR));
  ptr += len;
  return ChatString;
}
```

### 클라이언트 소켓 Packet Send

이제 실제 서버로 보내는 패킷을 구성하는 부분은 `FPacketHandler`와 `Protocol/P_PROTOCOL_PAKCET`이다.

채팅 메시지 패킷경우 다음처럼 구현했다. Type(일반 0), TextLen(텍스트 길이), Text(Fstring 채팅메시지)

```cpp
struct P_CHAT_PACKET
{
  uint8 Type;
  uint32 TextLen;
  FString Text;

  uint32 Size()
  {
    return sizeof(uint8) + sizeof(uint32) + TextLen;
  }
};
```

실제 패킷을 만들어서 보내는 코드는 다음과 같다. 언리얼의 `FBufferArchive`을 이용해서 버퍼를 할당하고, `MakePacket`함수가 보내는 패킷 buffer에 데이터를 입력해준다. 채팅 패킷은 다음처럼 구현했다.

```cpp
uint8* FPacketHandler::MakePacket(P_CHAT_PACKET& pkt, FBufferArchive& Ar)
{
  uint16 type = 4;
  uint16 size = sizeof(FPacketHeader) + pkt.Size();
  FPacketHeader header(type, size);

  Ar << header.type;
  Ar << header.size;
  Ar << pkt.Type;
  Ar << pkt.TextLen;
  Ar.Append((uint8*)pkt.Text.GetCharArray().GetData(), pkt.TextLen * sizeof(TCHAR));

  return Ar.GetData();
}
```

다음으로 서버에 FBufferArchive을 보내는 코드는 다음과 같다. Send함수로 buffer(ptr)시작주소에 count(길이)를 사용한다.

```cpp
int32 AMyFSocketActor::SendMessage(uint8* ptr, int32 count)
{
  int32 ByteSend = 0;

  if (_clientSocket)
  {
    while (ByteSend < count)
    {
      int32 _byteSent = 0;
      bool bResult = _clientSocket->Send(ptr, count, _byteSent);
      if (bResult)
        ByteSend += _byteSent;
      else
      {
        UE_LOG(LogTemp, Error, TEXT("Socket Actor SendMessage not Send !!!!"));
        break;
      }
    }
  }
  return ByteSend;
}
```

## 게임 플레이 영상

### 멀티 이동및 채팅

<iframe width="560" height="315" src="https://www.youtube.com/embed/VVc93B7wO2o" frameborder="0" allowfullscreen></iframe>

### 공격및 히트, 몬스터 타겟이동및 공격

<iframe width="560" height="315" src="https://www.youtube.com/embed/na8ZjzI547o" frameborder="0" allowfullscreen></iframe>

### 전체 영상

[**게임 클라이언트 코드 참조**](https://github.com/qornwh/BSGameClientUE5/tree/main)
