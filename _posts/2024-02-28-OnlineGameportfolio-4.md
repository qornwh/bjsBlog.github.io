---
title: 온라인게임 포트폴리오-4
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, boost, Unity, googleProtobuf, server, c++]
---

# 클라이언트 통신(Unity)

---

## 클라이언트 통신 목차

---

- 통신 함수
- 동작 구조
- PacketHandler, SocketComponent

### 통신 함수

소켓은 기본 c#의 `System.Net.Sockets` 사용하였고, `connection, recv, send`는 제공되는 비동기 함수를 사용했다.  
`connect`은 `ConnectAsync`으로 콜백함수로 연결 확인하고, `recv`는 `BeginReceive`함수로 `AsyncCallback` 콜백함수를 연결하고 콜백호출되면 다시 `BeginReceive`함수를 걸어준다, `send`는 `BeginSend`함수로 `AsyncCallback` 콜백함수를 연결한다. `send` 같은 경우는 보내고 호출되므로 확인용 같다.

```cpp
public void Connect(string addr, int port)
{
  IPAddress ipAddr = IPAddress.Parse(addr);
  IPEndPoint serverEndPoint = new IPEndPoint(ipAddr, port);
  SocketAsyncEventArgs args = new SocketAsyncEventArgs();
  args.Completed += OnConnectCompletedHandler;
  args.RemoteEndPoint = serverEndPoint;
  _socket.ConnectAsync(args);
}

public void OnConnectCompletedHandler(object obj, SocketAsyncEventArgs args)
{
  if (args.SocketError == SocketError.Success)
  {
    Debug.Log("서버와 연결 성공 !!!");
    _socket.BeginReceive(_recvBuffer.GetBuffer(), _recvBuffer.ReadPos(), _recvBuffer.FreeSize(), SocketFlags.None,
      new AsyncCallback(OnReciveHandler), null);
  }
  else
    Debug.Log("서버와 연결 실패 !!!");
}

public void OnReciveHandler(IAsyncResult result)
{
  // result.AsyncState; // => 이거 BeginReceive에서 마지막 파라미터에 들은 클래스로 캐스팅해서 확인가능. 그러나 일단제외 (IOCP 개념)
  int bytesRead = _socket.EndReceive(result);

  if (bytesRead > 0)
    _socket.BeginReceive(_recvBuffer.GetBuffer(), _recvBuffer.ReadPos(), _recvBuffer.FreeSize(), SocketFlags.None,
      new AsyncCallback(OnReciveHandler), null);
}

public void Send(IMessage msg, UInt16 id)
{
  byte[] buffer = PacketHandler.MakePacketHandler(msg, id);
  _socket.BeginSend(buffer, 0, buffer.Length, SocketFlags.None, new AsyncCallback(OnSendHandler), null);
}

public void OnSendHandler(IAsyncResult result)
{
  Debug.Log("데이터 전송 완료 !!!");
}
```

### 동작 구조

![unitySocket](/assets/img/online2/unitySocket.png)
_동작 구조_
위 그림처럼 `recv`가 들어오면 `queue`에 담기고 `SocketComponent`의 `update`가 동작될때 `pop`해서 `packet`을 가져온뒤 `id`에 맞게 처리된다. `send`인경우는 `queue`를 거치지 않고 바로 보내준다.

### PacketHandler, SocketComponent

`PacketHandler`는 `Packet heaer`구성, `Packet heaer` 체크및 파싱, `Send Buffer`에 Packet을 make하는 3가지로 구성되어 있다. `Packet heaer`같은 경우는 서버랑 동일하게 2(id)byte + 2(size)byte로 구성, `Google.Protobuf`사용해서 패킷이 만들어진다.

아래 코드를 확인해보면 다음과 같다.

```cpp
// 헤더사이즈만큼 패킷이 왔는지 확인.
public static bool CheckPacketHeader(PacketHeader header, int offset, int len)
{
  if (header == null)
    return false;

  int dataSize = len - offset;

  if (dataSize < 4)
    return false;

  if (dataSize < header.Size)
    return false;

  return true;
}

// 헤더 파싱
public static PacketHeader ParsePacketHandler(byte[] buffer, int offset)
{
  UInt16 id = BitConverter.ToUInt16(buffer, offset);
  UInt16 size = BitConverter.ToUInt16(buffer, offset + 2);
  PacketHeader header = new PacketHeader(id, size);

  return header;
}
// 패킷 버퍼에 복사
public static byte[] MakePacketHandler(IMessage pkt, UInt16 id)
{
  UInt16 pktSize = (UInt16) pkt.CalculateSize();

  byte[] headerBuffer = MakeHeaderPacketHandler(id, pktSize);
  byte[] pktBuffer = pkt.ToByteArray();

  byte[] sendBuffer = new byte[headerBuffer.Length + pktBuffer.Length];
  headerBuffer.CopyTo(sendBuffer, 0);
  pktBuffer.CopyTo(sendBuffer, headerBuffer.Length);

  return sendBuffer;
}
// 패킷 헤더
public static byte[] MakeHeaderPacketHandler(UInt16 pktId, UInt16 pktSize)
{
  // 일단 헤더는 2+2 = 4 바이트라 하드코딩 둔다.
  UInt16 packetSize = (UInt16) (pktSize + PacketHeader.Len);
  PacketHeader header = new PacketHeader(pktId, packetSize);
  byte[] headerArr = new byte[PacketHeader.Len];
  BitConverter.GetBytes(header.Id).CopyTo(headerArr, 0);
  BitConverter.GetBytes(header.Size).CopyTo(headerArr, 2);

  return headerArr;
}
```

`SocketComponent`는 `Scene`에 들어있다. 그래서 매 틱마다 한번씩 업데이트되며 그때 패킷을 저장해놓은 큐가 비어있지 않으면 pop한뒤 switch-case로 id별로 패킷을 읽어서 처리한다.

[**추가적인 클라이언트 참조 자료는 여기로**](https://theta08.github.io/rpggame/rpgGame01/)

[**Unity 코드 참조**](https://github.com/Theta08/RpgProject)

### 구현 및 기능

---

1. [**프로젝트 소개**](/bjsBlog.github.io/posts/OnlineGameportfolio-0)
2. [**네트워크 라이브러리 구현및 중요기능(CoreLib)**](/bjsBlog.github.io/posts/OnlineGameportfolio-1)
3. [**게임서버 구현및 중요기능(GameServer)**](/bjsBlog.github.io/posts/OnlineGameportfolio-2)
4. [**피격판정 기능(GameEngine)**](/bjsBlog.github.io/posts/OnlineGameportfolio-3)
5. [**클라이언트 통신(Unity)**](/bjsBlog.github.io/posts/OnlineGameportfolio-4)
