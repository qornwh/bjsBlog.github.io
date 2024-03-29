---
title: 온라인게임 포트폴리오-2
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, boost, Unity, googleProtobuf, server, c++]
---

# 게임서버 구현및 중요기능(GameServer)

---

## 게임서버 구현및 중요기능 목차

---

- Config 구조, JsonParser, GameSkill, init
- GameService서버, GameSession세션 기능
- GamePacket, Google.Protobuf 추가
- GameObjectInfo 구조
- GameRooom, GameMap 구조

### Config 구조, JsonParser, GameSkill, init

캐릭터 스킬정보같은 경우는 다음처럼 구현.

1. `config.json` 파일에 캐릭터 마다 스킬정보(범위, 공격력)이 설정된 파일을 json 파싱을 한다.
2. 스킬정보를 `GameSkill`에 설정한다.
3. 서버시작시 json파싱후 `GameSkill`에 초기화 한다.

### GameService서버, GameSession세션 기능

`GameService`, `GameSession`은 CoreLib의 Service, Session를 상속했다. 추가로 들어가는 기능들은 `GameService`에서 클라이언트가 `Accpet`될때 `GameSession`을 생성하는 기능과 `boost의 acceptor`에 연결시 등록하는 기능이 들어가고, `GameSession`에서는 `recv`된 `packet`을 헤더별로 분해하고 각각 맞는 행동을 하고 `GameRoom`에 전달하는 기능이 있다.

### GamePacket, Google.Protobuf 추가

`Google.Protobuf`같은 경우는 `vcpkg`를 사용해서 설치하였고, 프로젝트의 링크에 넣고, 추가종속성에 경로와, 필요한 lib파일을 추가해서 사용했다.

`GamePacket`은 `Google.Protobuf`로 만들어진 패킷을 분해하거나 조립하고, 헤더를 붙이는 기능이 들어간 클래스이다. 기능은 다음과 같다.

1. 패킷헤더로 recv된 패킷크기 확인 기능, 이후 패킷을 읽음
2. 패킷 파싱기능, recv된 패킷 buffer Array를 `Google.Protobuf`선언된 패킷으로 파싱하는기능
3. 패킷을 만드는기능, `Google.Protobuf`된 패킷정보를 buffer Array에 직렬화 하는 기능
4. 패킷 헤더를 만드는 기능, 패킷헤더를 만들면서 buffer Array를 초기화 한다

코드는 다음과 같이 구현

```cpp
static bool CheckPacketHeader(const boost::asio::mutable_buffer& buffer, PacketHeader* header, int32 offset, int32 len)
{
  if (header == nullptr)
    return false;

  int32 dataSize = len - offset;

  if (dataSize < sizeof(PacketHeader)) // 패킷헤더만큼 크기가 안될경우
    return false;

  if (dataSize < header->size) // 패킷헤더의 길이가 버퍼만큼의 크기가 안될경우
    return false;
  return true;
}

static bool ParsePacketHandler(google::protobuf::Message& pkt, const boost::asio::mutable_buffer& buffer, const int32 dataSize, int32& offset)
{
  const BYTE* payloadPtr = static_cast<BYTE*>(buffer.data()) + offset;
  const bool parseResult = pkt.ParseFromArray(payloadPtr, dataSize); // google::protobuf의 파싱이 되면 true
  if (parseResult)
  {
    offset += static_cast<int32>(pkt.ByteSizeLong());
    return true;
  }
  else
    return false;
}

static SendBufferRef MakePacketHandler(google::protobuf::Message& pkt, uint16 pktId)
{
  uint16 pktSize = pkt.ByteSizeLong();
  SendBufferRef sendBuffer = MakeHeaderPacketHandler(pktId, pktSize); // 패킷헤더

  if (pktSize > 0)
    pkt.SerializeToArray(&sendBuffer->Buffer()[sizeof(PacketHeader)], pktSize); // buffer array 직렬와

  return sendBuffer;
}

static SendBufferRef MakeHeaderPacketHandler(uint16 pktId, uint16 pktSize)
{
  const uint16 packetSize = pktSize + sizeof(PacketHeader);

  SendBufferRef sendBuffer = TLS_SendBufferManager->Open(packetSize);
  PacketHeader* header = reinterpret_cast<PacketHeader*>(sendBuffer->Buffer());
  header->id = pktId;
  header->size = packetSize;

  sendBuffer->Close(packetSize);

  return sendBuffer;
}
```

### GameObjectInfo 구조

먼저 `GameObjectInfo`는 플레이어, 몬스터의 상위 클래스이다. 그중 몬스터의 공격 쿨타임, 몬스터 랜덤 이동시 방향 전환 시간, 리스폰 시간등을 구현하기 위해 일정시간을 재는 `TickCounter`을 구현.
`TickCounter`의 전체적인 구조는 다음을 따른다.

1. tickCounter가 몇 tick동안 진행될값을 정한다. ex) 1초면 10 tick, 1tick당 100ms
2. 몬스터가 플레이어에게 공격을 한다고 할때, 처음 공격하고 딜레이가 있기때문에 그 시간동안은 공격만 해야된다. ex)공격 애니메이션이 1초면 그때동은 공격 모션만 보여줘야 되기 때문에 tick을 10으로잡고 몬스터 상태값을 변경하지 않는다.
3. 공격이 끝나면 공격tick은 끝나고 다른 행동을 한다.
   ![tick](/assets/img/online2/tick.png){: width="400" height="100%"}

`GameObjectInfo`에는 `GameRooom`에서 매 tick마다 한번씩 처리되는 `Update`함수 내에서 처리한다. 과정은 다음과 같다.
플레이어인 경우

1. attack == true
1. 일반공격, 스킬 코드 체크
1. `GameEngin`의 `Collision`으로 몬스터들과 충돌체크
1. 충돌된 몬스터 hp수정
   몬스터인 경우
1. 몬스터 상태체크
1. 상태마다 tick이 max값인경우 해당행동 실행 이동, 공격, 피격, 사망, 리스폰등
1. 공격시에는 `GameEngin`의 `Collision`으로 플레이어들과 충돌체크
1. 상태에 맞는 행동처리후 `packet`을 만들어서 `GameRoom`내의 모든 플레이어에게 전달.

### GameRooom, GameMap 구조

`GameRooom`의 사용용도는 맵 즉 필드에 몬스터가 이동과, 피격, 공격등을 일정시간마다 1번씩 업데이트를 하기 위함이다.(`GameRoom`은 1틱이 100ms로 잡았다.)
`GameRooom`이 한번씩 돌아가기위해 `boost::asio::strand`를 사용해서 구현. `strand`는 2개로 1개는 tick(100ms)처리용과 다른 1개는 `Service, Session`에서 들어온 처리된다. `boost::asio::strand`가 jobQueue의 역할을 한다.
`GameRooom`의 전체적인 구조는 다음을 따른다.

1. `Service, Session` 작업을 `GameRoom`으로 전달.
2. `boost::asio::strand`가 돌면서 작업을 순차적으로 처리.
3. `GameRoom`이 업데이트 될때 모든 유닛들이 업데이트 되면서 변경된사항을 패킷으로 만들어 `BroadCast`
   ![GameRoom](/assets/img/online2/GameRoom.png)

[**GameServer 코드 참조**](https://github.com/qornwh/GameServerProject/tree/main/GameServer)

### 구현 및 기능

---

1. [**프로젝트 소개**](/bjsBlog.github.io/posts/OnlineGameportfolio-0)
2. [**네트워크 라이브러리 구현및 중요기능(CoreLib)**](/bjsBlog.github.io/posts/OnlineGameportfolio-1)
3. [**게임서버 구현및 중요기능(GameServer)**](/bjsBlog.github.io/posts/OnlineGameportfolio-2)
4. [**피격판정 기능(GameEngine)**](/bjsBlog.github.io/posts/OnlineGameportfolio-3)
5. [**클라이언트 통신(Unity)**](/bjsBlog.github.io/posts/OnlineGameportfolio-4)
