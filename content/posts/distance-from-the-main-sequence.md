---
title: "Distance From the Main Sequence"
description: "메인 시퀀스로부터의 거리(distance from the main sequence)는 아키텍처 구조를 평가하는 몇 가지 메트릭중의 하나이다."
date: 2023-03-12T14:09:09+09:00
url: "/distance-from-the-main-sequence/"
tags: [software engineering]
---

메인 시퀀스로부터의 거리(distance from the main sequence)는 아키텍처 구조를 평가하는 몇 가지 메트릭중의 하나이다.
메인 시퀀스로부터의 거리는 불안정도와 추상도를 이용하여 계산하므로 먼저 불안정도와 추상도에 대해 알아보자.

## 추상도 (abstractness)

추상도는 (추상 클래스, 인터페이스 등의) 추상 아티팩트(abstract artifact)와 구상 아티팩트(concrete artifact, 구현체)의 비율,
즉 구현 대비 추상화 정도를 나타낸다. 다음은 추상도를 구하는 공식이다.

![추상도](/images/abstractness.png)
(M<sup>a</sup> 는 모듈에 있는 추상 요소 (인터페이스 또는 추상 클래스), M<sup>c</sup>는 구상 요소 (비 추상 클래스))

예를 들어 5,000 라인의 코드를 모두 main() 메서드에 구현한 애플리케이션의 분자는 1, 분모는 5,000 이므로 추상도는 0에 가깝다.

## 불안정도 (instability)

불안정도는 원심 커플링과 (구심 커플링 + 원심 커플링) 의 비율이다. 공식은 다음과 같다.

![불안정도](/images/instability.png)
(C<sup>e</sup>는 원심 커플링, C<sup>a</sup>는 구심 커플링)

여기에서 구심(afferent) 커플링은 (컴포넌트, 클래스, 함수 등의) 코드 아티팩트로 유입되는(incoming) 접속 수를,
원심(efferent) 커플링은 다른 코드 아티팩트로 유출되는(outgoing) 접속 수를 나타낸다.

불안정도는 코드베이스의 변동성(volatility)을 의미하므로 불안정도가 높은 코드베이스는 변경 시 커플링이 높아 더 깨지기 쉽다.
예를 들어, 여러 다른 클래스를 호출하는 클래스는 호출되는 메서드 중 하나라도 변경되면 호출하는 이 클래스 역시 잘못될 확률이 높다.

## 메인 시퀀스로부터의 거리

메인 시퀀스로부터의 거리는 불안정도와 추상도를 이용하여 계산한다. 공식은 다음과 같다.

![메인 시퀀스로부터의 거리](/images/distance_from_the_main_sequence.png)
(A는 추상도를, I는 불안정도)

추상도와 불안정도는 비율이므로 항상 0과 1 사이의 값이다.
따라서 다음과 같은 그래프로 표현할 수 있다.

![메인 시퀀스로부터의 거리 그래프](/images/distance_from_the_main_sequence_graph.png)

메인 시퀀스로부터의 거리가 메인 시퀀스에 가까울수록 추상도와 불안정도 사이에 균형을 잘 이룬다고 볼 수 있다.

오른쪽 위로 치우친 부분을 쓸모없는 구역(zone of uselessness), 반대로 왼쪽 아래로 치우친 부분을 고통스러운 구역(zone of pain) 이라고 부른다.
쓸모없는 구역에 속한 클래스는 추상화를 너무 많이해서 사용하기 어려운 코드이고,
고통스런 구역에 속한 클래스는 추상화를 거의 하지 않아 취약하고 관리하기 힘든 코드이다.

![쓸모없는 구역과 고통스러운 구역](/images/distance_from_the_main_sequence_graph2.png)
