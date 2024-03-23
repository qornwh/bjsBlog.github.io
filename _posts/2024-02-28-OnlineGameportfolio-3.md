---
title: 온라인게임 포트폴리오-3
author: bjs
date: 2024-03-01 00:00:00 +0800
categories: [Blogging, Portfolio]
tags: [온라인게임 개발기록, boost, Unity, googleProtobuf, server, c++]
---

# 피격판정 기능(GameEngine)

---

## 피격판정 기능 목차

---

- Vector2 클래스
- 2점 사이의 각도 구하기
- Collider, Shape 클래스

### Vector2 클래스

`Vector2` 클래스를 구현한 이유는 피격판정 즉 서버에서 오브젝트끼리 겹칠때 높이축는 신경쓰지 않고 가로, 세로축으로만 판정하기 위한 자료형이다. (추가로 변수가 x,y인 이유는 unity에서는 왼손좌표계라서 x,z 축이지만 오른손좌표계로 읽어와서 처리하기 때문이다.)

### 2점 사이의 각도 구하기

몬스터가 타겟(플레이어)으로 이동될때 몬스터방향벡터와 플레이어방향벡터간의 각도를 방식은 같다.

1. 플레이어, 몬스터의 좌표벡터를 구한다.
2. 두 좌표벡터의 차이의 벡터를 구한다.
3. 2번에서 구한 벡터와 정면벡터의 내적을 구한다. 여기서 정면벡터를 사용한 이유는 2벡터간 각을 구할수도 있지만 굳이 그 각도만큼 돌리기보다 글로벌좌표에서 회전한 각도를 바로 set하거나 현재 방향 각도와 정면차이랑 회전된 각도로 구할수 있다.
4. 외적의 z축값으로 음의 방향 양의 방향을 각도를 구한다.

![2점사이](/assets/img/online2/2점사이.png){: width="400" height="100%"}
_2점 사이의 각도 구하기_

```cpp
static float DotProduct(Vector2 a, Vector2 b)
{
  return a.X * b.X + a.Y * b.Y;
}

// 벡터 외적 공식 사용
static float crossProductAngle(Vector2 a, Vector2 b)
{
  return a.X * b.Y - b.X * a.Y;
}

// 벡터 크기
static float Magnitude(Vector2 vector)
{
  return static_cast<float>(std::sqrt(pow(vector.X, 2) + pow(vector.Y, 2)));
}

static Vector2 NormalizeVector(Vector2 vector)
{
  float mag = Magnitude(vector);

  if (mag != 0.0)
    return Vector2(vector.X / mag, vector.Y / mag);
  else
    return Vector2(0, 0);
}

// 2점 사이각 구한는 함수
static float CalculateAngle(Vector2& a, Vector2& b)
{
  Vector2 sub = b - a;
  //정면벡터를 쓴 이유는 기준으로 잡기 위해 마지막에 나온 각도를 더하는게 아닌 set을 하기때문
  Vector2 forward(1, 0);
  float dot = DotProduct(NormalizeVector(sub), forward);
  float radian = acos(dot);
  float angleRad = GameEngine::MathUtils::RadianToDegree(radian);

  if (crossProductAngle(forward, sub) > 0)
  {
    return angleRad;
  }
  else
  {
    return angleRad * (-1);
  }
}
```

### Collider, Shape 클래스

`Shape`클래스는 `Collier`가 원, 사각형에 대한 정보가 들어있다. 사각형인경우 width, height에 대한 벡터가 `Vertex`에 구현되어 있다. `Vertex`는 사각형 끼리 겹쳐진걸 확인할때 사용한다.

`Collider`클래스는 각 `Collier`끼리 현재 겹쳐진걸 기반으로 피격을 확인하는 용도로 구현. 겹쳐진걸 판단하는 Case는 3가지가 있는다. 각 Case마다 동작은 다음과 같다.

1. 원, 원

- 원의 중심점끼리의 길이보다 2개의 원의 반지름길이의 합보다 작으면 겹쳐진 상태이다.

2. 원, 사각형 or 사각형, 원

- 사각형을 기준으로 사각형 중점을 0,0으로 좌표변환과 그에맞게 원도 같이 좌표변환을 한다.
- 추가로 원은 1사분면으로 좌표변환 한다.
- 사각형의 x, y에 원의 반지름을 더하고 원의 좌표보다 크면 겹쳐진 상태이다.

3. 사각형, 사각형

- obb(Oriented Bounding Box) 분리축이론을 사용해서 구현.
- `Shape`클래스에는 `GetVertexs` 함수가 있다. 이함수로 width, height의 벡터를 구하는데 이렇게 하면 자연스럽게 1사분면 위치로 가게되고, 나중에 내적으로 구하면 기준벡터에 1사분면의 꼭지점으로 투영되기 때문이다.
- 투영시킨 1사분면 꼭지점 2개의 길이가 2점사이의 길이보다 작으면 교차되지 않은거고, 4번(각 사각형 가로 세로 벡터)다 통과시 교차가 된 것이다.

아래는 그림자료이다.  
![circleToCircle](/assets/img/online2/circleToCircle.png){: width="400" height="100%"}
_case 1 원, 원_
![circleToRect](/assets/img/online2/circleToRect.png){: width="400" height="100%"}
_case 2 원, 사각형 or 사각형, 원_
![rectToRect](/assets/img/online2/rectToRect.png){: width="400" height="100%"}
_case 3 사각형, 사각형_

[**GameEngine 코드 참조**](https://github.com/qornwh/GameServerProject/tree/main/GameEngine)

### 구현 및 기능

---

1. [**프로젝트 소개**](/posts/OnlineGameportfolio-0)
2. [**네트워크 라이브러리 구현및 중요기능(CoreLib)**](/posts/OnlineGameportfolio-1)
3. [**게임서버 구현및 중요기능(GameServer)**](/posts/OnlineGameportfolio-2)
4. [**피격판정 기능(GameEngine)**](/posts/OnlineGameportfolio-3)
5. [**클라이언트 통신(Unity)**](/posts/OnlineGameportfolio-4)
