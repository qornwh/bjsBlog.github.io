---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-0
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio2]
tags: [온라인게임 개발기록, epoll, unreal, server, c++]
---

# 리눅스의 Epoll과 언리얼 엔진으로 온라인 게임 개발

---

## 프로젝트 소개

---

- 개발 기간 : 2023.11.01 ~ 2024.2.10, 2024.3.3 ~ 2024.3.19 (약 4달)
- 개발 인원 : 1명
- 개발 환경
  - 서버 : 우분투 22.04(노트북 or Azure),
  - 클라이언트 : 윈도우 11
  - unrealEngine(v5.1.1)

## 전체 영상

<iframe width="800" height="515" src="https://www.youtube.com/embed/NIEy5bpEhBk" frameborder="0" allowfullscreen></iframe>
<iframe width="800" height="600" src="https://www.youtube.com/embed/ymCHGSQRh6s" frameborder="0" allowfullscreen></iframe>

## Source Code

- [**Server**](https://github.com/qornwh/BSGameServer)
- [**Client**](https://github.com/qornwh/BSGameClientUE5)

## 게임 서비스 구조 - 서버(리눅스 Epoll)

---

![서버구조](/assets/img/epoll_unreal_서버구조.png)

_서버구조 간략도_

위 그림과 같이 게임서버는 io를 담당하는 Service에서는 클라이언트로부터 packet을 받고 현재 클라이언트가 속한 Room에서 Queue 채팅, 공격, 이동등의 메시지를 넣는다. 다음에 Room에서는 메시지가 존재하면 해당하는 패킷의 채팅, 공격, 이동을 처리하면서 각 클라이언트로 전송한다.

## 게임 서비스 구조 - 클라이언트(언리얼 엔진)

---

![클라구조](/assets/img/epoll_unreal_클라구조.png)

_클라이언트구조 간략도_

위 그림과 같이, 게임 클라이언트에서는 게임시작시 언리얼의 FSocket으로 게임서버와 연결하고, 플레이어(자신)이 이동, 공격, 채팅등 패킷을 FSocket으로 서버로 메시지를 보낸다. 다른 플레이어와 몬스터들의 이동, 공격, 채팅등은 FSocket에서 패킷을 읽어와서 컴포넌트에서 처리된다.

## 게임 서비스 제작및 구현

---

1. [**프로젝트 소개**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-0)
2. [**Epoll을 이용한 소켓 통신**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-1)
3. [**게임 패킷 구현 및 멀티 스레드**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-2)
4. [**게임 기능 구현**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-3)
5. [**언리얼 엔진에서의 소켓 통신및 플레이 영상**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-4)
6. [**vscode 설정, makefile**](/bjsBlog.github.io/posts/OnlineGame-EpollServer-5)
