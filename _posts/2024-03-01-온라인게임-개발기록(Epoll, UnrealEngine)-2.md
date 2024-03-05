---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-2
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal]
---

# 게임 패킷 구현

## 패킷 구성

패킷은 패킷 헤더 + 패킷 내용으로 구성된다. 패킷 헤더에는 id(2byte) + 패킷 크기(2byte)로 구성되고 패킷 내용에는 이동, 채팅, 공격 메시지에 대한 내용이 담겨있다.
![Packet](/images/Packet.png)

## 패킷 Read

![ReadBuffer](/images/ReadBuffer.png)
먼저 Tcp로 통신하게 되므로 패킷이 오는 순서는 보장되어 있지만, 중간이 나눠져서 올 수 있다. 예를들면 다음과 같다. ex) 클라 50보냄 => 서버 25받음, 서버 25받음.이런식으로 데이터를 받을경우 buffer에 데이터를 쌓아둔뒤에 패킷 크기만큼 읽어서 처리한다.

## 패킷 Send

![SendBuffer](/images/SendBuffer.png)
SendBuffer는 항상 메모리를 할당해서 버퍼를 만드는게 아닌 풀링을 해서 구현하였다. 위 그림처럼 SendBufferManager는 전역에 하나 만들어 두고, vector로 SendBufferChunk가 있다. SendBufferChunk는 SendBuffer가 패킷크기마다 다르기때문에 일정범위의 1byte Array를 만들고 패킷 사이즈만큼 Senbuffer를 만들고 한번 전부다 라이팅되면 다시 전역에 push하는 방법으로 구현하였다.

[**네트워크 라이브러리 코드 CoreLib폴더 참조**](https://github.com/qornwh/BSGameServer/tree/main/CoreLib)
