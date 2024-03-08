---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-3
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal]
---

# 게임 기능 구현

## 게임 서비스

![GameService](/assets/img/GameService.png)

_GameService_
Service : 게임서버,
Session : 게임 클라이언트의 정보

**게임 서비스는 다음처럼 작동되도록 구현했다.**

1. Serivce 초기화
2. 필요한만큼 스레드 생성
3. 각 스레드에서 동일한 작업 진행(Dispatch)
   1. 각 스레드에서 네트워크 io받아서 Session에 맞는 패킷 분해
   2. 패킷 id확인후 맞는 처리
   3. 현재 클라이언트가 있는 GameRoom에 Job Queue 처리
   4. GameRoom의 Session별로 BroadCast
4. 각 스레드에서 동일한 작업 진행(GameRoom Task)
   1. 몬스터가 deSpawn이면 다시 Spawn.
   2. 몬스터의 랜덤 이동
   3. 몬스터가 피해를 입고 Target(player)가 monsterMap에 있으면 target방향으로 이동.
   4. 몬스터의 상태 GameRoom의 Session별로 BroadCast

### 게임 멀티 스레드

```cpp
void RunTask(BS_GameServerServiceRef serverRef)
{
  while (1)
  {
    serverRef->Dispatch();
    shared_ptr<BS_GameRoom> room = GBSRoomManger->getRoom(0);
    if (room != nullptr)
    {
      room->RoomTask();
    }
  }
}

int main()
{
  BS_GameServerServiceRef serverRef = make_shared<BS_GameServerService>();
  ASSERT_CRASH(serverRef->Start());
  // 스레드 생성
  ThreadManager::GetInstance().CreateThread([&serverRef]() { RunTask(serverRef); });
  RunTask(serverRef);
  // ...
}
```

main 함수에서 io와 GameRoom task를 처리할 스레드를 만들고, 메인스레드와 다른스레드 모두 같은 작업을 진행한다.

### 게임 서비스 Dispatch

```cpp
void ServerService::Dispatch()
{
  // ...
  for (int i = 0; i < eventCount; i++)
  {
    if (isServerFd(i))
      AcceptClient(i);
    else
      ReadClient(i);
  }
}

///
void ServerService::ReadClient(int idx)
{
  int clientFd = _serverSock->getEpollEvent(idx).data.fd;

  SessionRef session = GetSession(clientFd);
  if (session != nullptr)
  {
    // atomic으로 compare_exchange_strong을 한 이유.
    // 멀티스레드에서 동시에 읽어오는 이슈가 있어서 사용. fd읽는 도중에 또 같은 fd를 읽어버림 !!!
    bool isReading = session->IsReading();
    if (!isReading && session->CheckReading(isReading))
    {
      bool isRecv = session->ReciveMessage();
      session->OffReading();
      if (!isRecv)
        ReleaseSession(session);
    }
  }
}

// EpollFD.h
class EpollFD : public enable_shared_from_this<EpollFD>
{
  // ....
  epoll_event &getEpollEvent(int i)
  {
    // 사실상 _epollEvents 리스트는 연결된 socket 개수 리스트 이기때문에 이걸로
    // 실제 socket fd 찾기는 불가 다른 map으로 만듬, _epollEvents[i].data.fd로 socket fd 접근가능
    return _epollEvents[i];
  }
  // ...

protected:
  struct epoll_event _epollEvents[MAX_CLIENT];
};
```

Dispatch 함수에서 서버와 클라이언트를 Accept와 Read를 작동하게 된다. 함수를 보게되면 i 인덱스를 파라미터로 받게 되는데 저 i인덱스는 서버소켓의 `_epoll_event`로 받아온 socket 리스트이다. 추가적으로 `멀티스레드 이슈`로 read할때 epoll에 버퍼가 차있으면 1번 읽을수 있을때까지 신호가 있는데 아직 `recv`함수를 호출하지 않은 상태이면 동시에 접근이 가능해서 CAS로 동시접근이 되지 않도록 처리했다.

### 게임 서비스 RecvMessage

```cpp
bool Session::ReciveMessage()
{
  int32 strLen = recv(_socketFd, _recvBuffer.WritePos(), _recvBuffer.FreeSize(), 0);

  if (strLen <= 0)
  {
    Disconnect();
    return false;
  }
  else
  {
    if (!_recvBuffer.OnWrite(strLen))
    {
      Disconnect();
      return false;
    }

    int32 dataSize = _recvBuffer.DataSize();
    BYTE *buffer = _recvBuffer.ReadPos();
    int32 processLen = OnRecv(buffer, dataSize);
    if (!_recvBuffer.OnRead(processLen) || processLen <= 0 || dataSize < processLen)
    {
      Disconnect();
      return false;
    }

    _recvBuffer.Clean();
  }
  return true;
}

int32 BS_GameSession::OnRecv(BYTE *buffer, int32 len)
{
  int32 useLen = 0;
  while (true)
  {
    // ...
    PacketHeader *header = reinterpret_cast<PacketHeader *>(&buffer[useLen]);
    // ...
    HandlePacket(&buffer[useLen], header->size);
    // ...
  }

  return useLen;
}
```

`recv`함수로 `readbuffer`로 패킷을 가져오고, `size ==0`이면 종료된 클라이언트이므로 `disconnect`를 호출한다. 여기서 제일 중요한 함수는 `OnRecv`인데 여기서 `PacketHeader`를 분리하고 `HandlePacket`함수로 패킷메시지 를 분리해 낸다.

```cpp
void BS_GameSession::HandlePacket(BYTE *buffer, int32 len)
{
  PacketHeader *header = reinterpret_cast<PacketHeader *>(buffer);
  uint16 id = header->id;
  switch (id)
  {
  case 1:
  {
    int offset = sizeof(PacketHeader);

    uint16 *type = PacketUtils::ReadBufferPtr<uint16>(buffer, offset);
    uint16 *nameLen = PacketUtils::ReadBufferPtr<uint16>(buffer, offset);

    BYTE *name = PacketUtils::ReadBufferStr(buffer, offset, *nameLen);
  // ...
  }}}

///
template <typename T>
T *PacketUtils::ReadBufferPtr(BYTE *buffer, int &offset)
{
  int _offset = offset;
  offset += sizeof(T);
  T *temp = reinterpret_cast<T *>(&buffer[_offset]);
  return temp;
}

BYTE *PacketUtils::ReadBufferStr(BYTE *buffer, int &offset, uint32 len)
{
  int tempOffset = offset;
  offset += len;
  return &buffer[tempOffset];
}
```

`HandlePacket`함수는 실제 `buffer` 주소 위치를 참조해서 읽어내는데 읽게 도와주는게 `ReadBufferPtr, ReadBufferStr` 함수다. 2함수 모두 시작위치 주소를 리턴으로 넘기고, offset이 size만큼 더하므로써 다음위치를 바라보도록 구현했다.

### 게임 패킷 핸들러 구현

이제 실제 게임에서 보내는 패킷을 구성하는 부분은 `BS_PacketHandler`와 `BS_Packet`이다. `BS_Packet`에 보낼 패킷에 어떤내용들을 보낼지 구성되어 있다.

간단한 채팅 메시지인 경우 다음처럼 구현했다. 보내는 유저(Code), Type(일반 0), TextLen(텍스트 길이), Text(string 첫 주소)

```cpp
struct BS_C_CHAT
{
  uint32 Code;
  uint8 Type;
  uint32 TextLen;
  BYTE *Text;

  int32 size()
  {
    return sizeof(uint8) + sizeof(uint32) + sizeof(uint32) + TextLen;
  }
};
```

실제 패킷을 만들어서 보내는 코드는 다음과 같다. `MakeSendBuffer`함수가 패킷 헤더랑 크기를 Buffer로 할당해주는 함수이고, `MakePacket`함수가 보내는 패킷 buffer에 데이터를 입력해준다.

```cpp
  // 유저 채팅 메시지
static SendBufferRef MakePacket(BS_Protocol::BS_C_CHAT &pkt)
{
  const uint16 dataSize = pkt.size();
  SendBufferRef sendBuffer = MakeSendBuffer(dataSize, 4);

  BufferWrite bw(sendBuffer->Buffer() + sizeof(PacketHeader), dataSize);
  bw.Write(&pkt.Code);
  bw.Write(&pkt.Type);
  bw.Write(&pkt.TextLen);
  bw.Write(pkt.Text, pkt.TextLen);
  return sendBuffer;
}

static SendBufferRef MakeSendBuffer(uint16 dataSize, uint16 pktId)
{
  const uint16 packetSize = dataSize + sizeof(PacketHeader);

  SendBufferRef sendBuffer = GSendBufferManger->Open(packetSize);
  PacketHeader *header = reinterpret_cast<PacketHeader *>(sendBuffer->Buffer());
  header->id = pktId;
  header->size = packetSize;

  sendBuffer->Close(packetSize);

  return sendBuffer;
}
```

### 게임 맵 구현

![GameMap](/assets/img/GameMap.png){: width="300" height="400"}

_GameMap_
게임 룸을 구현하기 먼저 맵의 크기가 필요하다. 맵의 크기가 있어야 몬스터들이 이동할때 맵을 나가지 않도록 만들 필요가 있다. 맵은 전체크기와 몬스터가 이동할수 있는 구역을 빨간색 구역을 나누었다. 단 게임 맵 안에 벽은 없다.

### GameRoom 구현

![GameRoom](/assets/img/GameRoom.png){: width="400" height="250"}

_GameRoom_
위 그림처럼 각 던전에서는 다른 던전의 몬스터 플레이어는 확인하지 않고, 각 룸에서 공격 이동등은 던전에 포함된 몬스터, 플레이어끼리 메시지를 주고 받는다. 즉 중요한점은 룸끼리는 각각 따로 작동된다.

그래서 `GameRoom`은 `JobQueue`를 상속받고, 현재 필드의 몬스터 정보, 일정시간마다 한번씩 발생하는 `tick`함수, `GameMap`정보, 등으로 구성했다.

### GameRoom Tick 함수구현

리눅스에 시스템이 경과된 시간을 가져오는 함수가 없어서 구현해서 사용했다. `timespec`는 초와 나노초 단위를 기록하고, `clock_gettime`함수로 수행시간을 측정한다.

```cpp
static uint64 GetTickCount64_OS()
{
  uint64 tick;
  struct timespec tp;
  clock_gettime(CLOCK_MONOTONIC, &tp);
  tick = (tp.tv_sec * 1000ull) + (tp.tv_nsec / 1000ull / 1000ull);
  return tick;
}
```

### GameRoom 몬스터 구현

몬스터와 플레이어에 대한 정보는 `BS_Player`클래스에 정의되어 있다.
몬스터 행동처리는 다음과 같이 구현했다.

1. 기본 이동 : 방향만 mt19937으로 난수생성으로 2초마다 한번 업데이트, `GameMap`범위를 나가진 않는다.
2. 플레이어로 부터 피격 : 공격해야될 target(player)이 설정됨
3. target(player)로 이동 : 몬스터 방향과 플레이어 방향과 위치 정보가 있으므로 2벡터 사이의 각을 외적으로 구한다음에 그방향으로 바로 이동되도록 구현, 단 플레이어가 `GameMap`에 있으면 따라가고, 없으면 기본이동처리를 한다.
4. 공격 : 일정범위에 있으면 다시한번 2벡터 사이의 각을 외적으로 구한다음에 공격 `packet`을 모든 클라이언트들에게 보낸다.
5. 리스폰 : 몬스터 체력이 0이 되면 사망처리 및 사망 `packet`을 모든 클라이언트들에게 보내고, 다시 Tick함수가될때 사망처리된 몬스터들은 모두 리스폰시키고, `packet`을 모든 클라이언트들에게 보낸다.

### target(player)로 이동

target방향으로 이동하기 위해서는 2벡터 사이의 거리를 구해야된다. 아래 그림에서는 A를 대상으로 한다.

![2점사이](/assets/img/2점사이.png){: width="400" height="250"}

2 벡터 간의 각도를 계산하기 위해 다음처럼 구현했다.

1. B벡터 - A벡터로 타겟방향으로 가는 벡터를 구함
2. 정면벡터와 타겟벡터 내적을 한뒤 acos함수로 각도 `radian`구함.
3. `radian`을 `angle`로 변환뒤
4. 정면벡터와 타겟벡터를 외적한뒤 양수 음수 판단.
5. 음수면 `angle`\* -1을 한다.

실제 함수는 다음처럼 구현했다.

```cpp
// 벡터 내적 공식 사용
static double dotProduct(float x1, float y1, float x2, float y2)
{
  return x1 * x2 + y1 * y2;
}
// 벡터 외적 각 +- 판단
static double crossProductAngle(double x1, double y1, double x2, double y2)
{
  return x1 * y2 - x2 * y1;
}
// 벡터 크기
static double magnitude(float x, float y)
{
  return sqrt(x * x + y * y);
}
// 벡터 정규화
static void normalizeVector(float &x, float &y)
{
  float mag = magnitude(x, y);

  // 벡터의 크기가 0이 아닌 경우에만 정규화 수행
  if (mag != 0.0)
  {
    x /= mag;
    y /= mag;
  }
}
// 라디안을 각도로 변환하는 함수
static double radianToDegree(double radian)
{
  return radian * 180.0 / M_PI;
}

// 두 벡터 간의 각도를 계산하는 함수 (라디안 단위)
static float calculateAngle(float x1, float y1, float x2, float y2)
{
  float x = x2 - x1;
  float y = y2 - y1;
  float x_forward = 1;
  float y_forward = 0;
  normalizeVector(x, y);
  double dot = dotProduct(x_forward, y_forward, x, y);

  double radian = acos(dot);
  float angleRad = radianToDegree(radian);

  if (crossProductAngle(x_forward, y_forward, x, y) > 0)
  {
    return angleRad;
  }
  else
  {
    return angleRad * (-1);
  }
}
```

[**게임 서비스 코드 GameBS폴더 참조**](https://github.com/qornwh/BSGameServer/tree/main/GameBS)
