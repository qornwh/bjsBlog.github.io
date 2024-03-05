---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-3
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, epoll, unreal]
---

## 게임 기능 구현

## 게임 룸 구현

![GameRoom](/images/GameRoom.png){: width="400" height="250"}
_GameRoom_
위 그림처럼 각 던전에서는 다른 던전의 몬스터 플레이어는 확인하지 않고, 각 룸에서 공격 이동등은 던전에 포함된 몬스터, 플레이어끼리 메시지를 주고 받는다.

![GameMap](/images/GameMap.png){: width="300" height="400"}
_GameMap_
게임 룸을 구현하기 먼저 맵의 크기가 필요하다. 맵의 크기가 있어야 몬스터들이 이동할때 맵을 나가지 않도록 만들 필요가 있다. 맵은 전체크기와 몬스터가 이동할수 있는 구역을 빨간색 구역을 나누었다.

[**게임 서비스 코드 CoreLib폴더 참조**](https://github.com/qornwh/BSGameServer/tree/main/GameBS)
