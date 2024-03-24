---
title: 온라인게임 포트폴리오
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, boost, Unity, googleProtobuf, server, c++]
---

# 온라인게임 포트폴리오/서버

---

## 프로젝트 소개

---

- 개발 기간 : 2023.12.15 ~ 2024.3.10 (약 3달)
- 개발 인원 : 2명 (서버 1, 클라이언트 1)
- 개발 환경
  - 서버 : 윈도우 11
  - boost(v1.83.0)
  - google::protobuf(v3.21.12)
  - 클라이언트 : 윈도우 11
  - unity(v2021.3)

### 시연 영상

---

<iframe width="800" height="515" src="https://www.youtube.com/embed/WXoosvnCQw4" frameborder="0" allowfullscreen></iframe>
<iframe width="800" height="515" src="https://www.youtube.com/embed/k3-c4AiTmxs" frameborder="0" allowfullscreen></iframe>
<iframe width="800" height="515" src="https://www.youtube.com/embed/6-NsowB52Xw" frameborder="0" allowfullscreen></iframe>

### Source Code

- [**Server**](https://github.com/qornwh/GameServerProject)
- [**Client**](https://github.com/Theta08/RpgProject)

### 제작 과정

---

처음 개발을 목표는 `어떻게 많은 유저들이 동시에 연결되어 끊김 없이 게임을 즐길 수 있을까?` 입니다. 실제 운영되고 있는 게임서버에는 어떤식으로 처리되어서 대규모 1000명이 넘는 유저들이 한 공간에서 보여질수 있다는 호기심이 프로젝트의 시작이었습니다. 그래서 지인 1명과 같이 개발을 진행했습니다.

**개발 진행**

1. 먼저 서버와 클라이언트간의 tcp소켓을 연결할수 있는 Boost.asio 라이브러리 준비, CoreLib 프로젝트 생성
2. 소켓 서버는 Tcp프로토콜로 설정, read, write를 비동기 함수로 구현.
3. 공유자원 동시접근을 막기 위해 lock처리 구현(mutex사용)
4. 서버와 클라이언트에 대한 Service, Session 클래스 구현, GameServer 프로젝트 생성
5. Tcp 패킷을 위한 google.protobuf사용및 read, write buffer 구현
6. 플레이어, 몬스터정보는 \*info.class에 구현
7. 피격판정, 콜리전이 교차되었는지 판단하는 GameEngin 프로젝트 생성
8. 한 맵에서 몬스터 ai처리와 피격판정을 하는 GameRoom구현

### 사용 라이브러리

---

[boost](https://www.boost.org/)  
[Google Protobuf](https://protobuf.dev/)

## 구현 및 기능

---

1. [**프로젝트 소개**](/bjsBlog.github.io/posts/OnlineGameportfolio-0)
2. [**네트워크 라이브러리 구현및 중요기능(CoreLib)**](/bjsBlog.github.io/posts/OnlineGameportfolio-1)
3. [**게임서버 구현및 중요기능(GameServer)**](/bjsBlog.github.io/posts/OnlineGameportfolio-2)
4. [**피격판정 기능(GameEngine)**](/bjsBlog.github.io/posts/OnlineGameportfolio-3)
5. [**클라이언트 통신(Unity)**](/bjsBlog.github.io/posts/OnlineGameportfolio-4)
