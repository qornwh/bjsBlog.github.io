---
title: 온라인게임 개발기록(Epoll, UnrealEngine)-5
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio2]
tags: [온라인게임 개발기록, unreal, server, c++]
---

# vscode 설정, makefile

---

### vscode 설정

---

vscode를 사용한 이유는 초기에 visual studio로 원격접속으로 개발을 했지만, 멀티스레드에서 디버깅을 하면 계속 프로그램이 다운되는 문제가 있어. vscode를 사용하기로 했다. 그래서 vscode 디버깅은 `Task`와 `Launch`를 설정했는데. 코드는 다음과 같다.

**Launch**
Launch파일은 간단하게 miDebuggerPath디버깅할 gdb path를 주고, program은 현재 실행할 파일 `preLaunchTask`이 실행할 `Task`의 이름을 설정.

```json
{
  "configurations": [
    {
      "name": "빌드용",
      "type": "cppdbg",
      "request": "launch",
      "program": "${fileDirname}/${fileBasenameNoExtension}",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "MIMode": "gdb",
      "setupCommands": [
        {
          "description": "gdb에 자동 서식 지정 사용",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        },
        {
          "description": "디스어셈블리 버전을 Intel(으)로 설정",
          "text": "-gdb-set disassembly-flavor intel",
          "ignoreFailures": true
        }
      ],
      "preLaunchTask": "C/C++: g++ 활성 파일 빌드",
      "miDebuggerPath": "/usr/bin/gdb"
    }
  ]
}
```

**Task**
Args에 makeFile과 동일하게 c++버전, 같이 컴파일될 cpp, h파일을 넣고 commad + args로 실행된다.

```json
{
  "tasks": [
    {
      "type": "cppbuild",
      "label": "C/C++: g++ 활성 파일 빌드",
      "command": "/usr/bin/g++",
      "args": [
        "-fdiagnostics-color=always",
        "-std=c++17",
        "-Wall",
        "-fno-strict-aliasing",
        "-g",
        "${fileDirname}/BS_GameRoom.cpp",
        "${fileDirname}/BS_GameSession.cpp",
        "${fileDirname}/BS_PacketHandler.cpp",
        "${fileDirname}/BS_Pch.cpp",
        "${fileDirname}/BS_GameServerService.cpp",
        "${fileDirname}/BS_Packet.cpp",
        "${fileDirname}/main.cpp",
        "${fileDirname}/BS_GameRoomManger.cpp",
        "${fileDirname}/BS_MapInfo.cpp",
        "${fileDirname}/BS_Player.cpp",
        "${workspaceFolder}/CoreLib/CoreGlobal.cpp",
        "${workspaceFolder}/CoreLib/EpollFD.cpp",
        "${workspaceFolder}/CoreLib/Pch.cpp",
        "${workspaceFolder}/CoreLib/Session.cpp",
        "${workspaceFolder}/CoreLib/LockUtils.cpp",
        "${workspaceFolder}/CoreLib/ServerSock.cpp",
        "${workspaceFolder}/CoreLib/Job.cpp",
        "${workspaceFolder}/CoreLib/RecvBuffer.cpp",
        "${workspaceFolder}/CoreLib/SocketUtils.cpp",
        "${workspaceFolder}/CoreLib/CoreTLS.cpp",
        "${workspaceFolder}/CoreLib/Service.cpp",
        "${workspaceFolder}/CoreLib/JobQueue.cpp",
        "${workspaceFolder}/CoreLib/SendBuffer.cpp",
        "${workspaceFolder}/CoreLib/ThreadManager.cpp",
        //"${fileDirname}/**.cpp",       // 갑자기 안되는 이유는 모르겠다. 지금까지 잘써왔는데...
        //"${workspaceFolder}/CoreLib/**.cpp", // 갑자기 안되는 이유는 모르겠다. 지금까지 잘써왔는데...
        "-o",
        "${fileDirname}/${fileBasenameNoExtension}"
      ],
      "options": {
        "cwd": "/usr/bin"
      },
      "problemMatcher": ["$gcc"],
      "group": "build",
      "detail": "디버거에서 생성된 작업입니다."
    }
  ],
  "version": "2.0.0"
}
```

### makeFile

---

makefile 사용이유는 에디터가아닌 실행파일로 빌드하기 위해서 사용. `CoreLib`, `GameBS`를 나눠서 빌드.

## 게임 서비스 제작및 구현

---

1. [**프로젝트 소개**](/posts/OnlineGame-EpollServer-0)
2. [**Epoll을 이용한 소켓 통신**](/posts/OnlineGame-EpollServer-1)
3. [**게임 패킷 구현 및 멀티 스레드**](/posts/OnlineGame-EpollServer-2)
4. [**게임 기능 구현**](/posts/OnlineGame-EpollServer-3)
5. [**언리얼 엔진에서의 소켓 통신및 플레이 영상**](/posts/OnlineGame-EpollServer-4)
6. [**vscode 설정, makefile**](/posts/OnlineGame-EpollServer-5)
