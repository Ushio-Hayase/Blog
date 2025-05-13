---
title: 테트리스를 깨는 AI 제작 - 4 [역전파 알고리즘 총정리]
published: 2025-05-14
description: '테트리스를 스스로 깨는 강화학습 에이전트 CUDA부터 개발하기'
tags: ["C++","AI", "CUDA" ,"ReinforceLearning"]
category: 'Project'
draft: false 
lang: 'ko'
---

## 프로젝트 개요

AI에 대해서는 예전부터 관심이 있었고 C++과 rust같은 로우레벨 프로그래밍 언어도 개인적으로 좋아합니다.

그래서 이번 프로젝트는 C++와 CUDA를 활용해 외부 라이브러리 없이 테트리스 게임을 직접 구현하고,
이 게임을 스스로 플레이하며 최고 점수를 노리는 강화학습 에이전트를 처음부터 만들어보는 과정을 기록하는 것을 목표로 할 것입니다.

로우레벨 언어와 인공지능, 그리고 GPU 프로그래밍에 관심이 많은 학부생으로서, 이미 잘 만들어진 라이브러리나 프레임워크에 의존하지 않고
처음부터 모든 것을 직접 설계하고 구현해보는 경험을 통해, 진짜로 시스템이 어떻게 돌아가는지 깊이 이해하고 싶습니다.

또한, GPU의 병렬 연산 능력을 실제로 활용해보며, 이론으로만 배웠던 개념들이 실제 코드와 하드웨어에서 어떻게 동작하는지 체험하고자 합니다.

이 프로젝트를 통해 배우고자 하는 가장 큰 목표는, 강화학습의 핵심 원리와 GPU 프로그래밍의 실전 기술을 내 손으로 직접 구현하며 익히는 것입니다.

테트리스라는 익숙한 게임을 스스로 만들고, 그 위에서 동작하는 에이전트를 설계하면서, 상태 공간과 행동 집합, 보상 함수 설계의 중요성을 느낄 것입니다.

또한, CUDA를 활용해 대량의 시뮬레이션을 병렬로 처리하는 과정에서, GPU 메모리 관리나 커널 최적화와 같은 실전적인 문제들을 직접
해결해보고자 합니다.

라이브러리 없이 신경망이나 알고리즘을 처음부터 구현하는 과정에서, 평소에는 잘 느끼지 못했던 컴퓨팅 자원의 한계나, 병렬 연산의 어려움도
경험할 수 있을 것이라 기대하고 있습니다.

어려움도 많겠지만 이런 난관들을 직접 부딪히고 해결해가는 과정에서, 단순히 결과만 얻는 것이 아니라, 문제를 분석하고 해결책을 찾아가는 과정
자체가 큰 성장의 기회가 될 것이라고 생각합니다.

## 오늘 한 것

역전파 수학적 정의 알아보기

행렬 형태로 확장해서 알아보기

행렬의 미분 형태로 확장해서 알기

코드로 구현하기

## 역전파 알고리즘 정리

일단 이 블로그에서는 기본적으로 고등학교 수준의 미적분(다항함수의 미분)과 스칼라, 벡터, 행렬에 대한 개념은 알고있다고 가정하고 내용을 설명하겠습니다.

먼저 저희가 고등학교때 배웠던 미적분의 개념을 떠올려보면 다음과 같습니다.

$$\frac{dy}{dx} = \lim_{h\rightarrow 0} \frac{f(x+h)-f(x)}{h}=f^\prime(x)$$

이것은 **스칼라 정의역의 원소**를 스칼라 **공역에 있는 원소**에 대응시키는 함수 $f$에 대한 미분으로 볼 수 있습니다.

---

이제 **입력이 스칼라**이고 **출력이 스칼라**인 함수로 바꿀 수 있습니다.

이 함수는 당연하겠지만 한 스칼라에는 하나의 스칼라만 미분 가능하기때문에 원하는 변수만 취급하고 다른 변수는 상수로 취급하는 편미분을 이용해야합니다.

여기서 추가로 저희는 표기법을 알아야합니다.

행렬 미분에서 표기법은 2가지가 존재합니다.

- 분자 표기법 : 미분 대상(분자)의 차원을 행으로, 미분 변수(분모)의 차원을 열로 배치합니다.
- 분모 표기법 : 미분 변수(분모)의 차원을 행으로, 미분 대상(분자)의 차원을 열로 배치합니다.

두 표기법은 전치 관계에 있어서 주의해서 봐야합니다.

벡터-스칼라 미분으로 돌아가서 이걸 이제 분모기준 표기법으로 표기하면 각 $y$에 대해 벡터 $\bold{x}$의 원소로 편미분 한 것과 같으니
다음과 같습니다.

$$\bold{x}= \begin{bmatrix} x_1 \\ x_2 \\ ...\\ x_n \end{bmatrix}, f(\bold{x})= y, f^\prime(\bold{x})=\begin{bmatrix} \frac{\partial y}{\partial x_1}\\ \frac{\partial y}{\partial x_2} \\...\\\frac{\partial y}{\partial x_n} \end{bmatrix} $$

벡터-스칼라 미분도 알아봤으니 저희는 이제 스칼라-벡터 함수도 미분을 할 수 있습니다.

그럼 함수의 정의와 미분 결과는 다음과 같습니다.

$$\bold f(x) = \begin{bmatrix}f_1(x)\\f_2(x)\\...\\f_m(x)\end{bmatrix}, \frac{\partial \bold y}{\partial x}=\bold f^\prime(x)=\begin{bmatrix}\frac{\partial y_1}{\partial x}\\ \frac{\partial y_2}{\partial x} \\ ... \\ \frac{\partial y_m}{\partial x} \end{bmatrix}$$

---

이 생각을 더 확장해서 이제 저희는 **입력도 벡터**이고 **출력도 벡터**인 함수를 생각할 수 있습니다.

그러면 그 결과는 행렬이 나오고 그 행렬을 자코비안 행렬이라 부르며 보통 벡터를 다른 벡터로 변환시키므로 선형변환입니다.

$$\bold x = \begin{bmatrix} x_1 \\ x_2 \\ ...\\ x_n \end{bmatrix},  \bold y = \begin{bmatrix} y_1 \\ y_2 \\ ...\\ y_n \end{bmatrix}, \frac{\partial \bold y}{\partial \bold x} = \begin{bmatrix} \frac{\partial y_1}{\partial x_1} & \frac{\partial y_1}{\partial x_2} & \cdots & \frac{\partial y_1}{\partial x_n} \\ \frac{\partial y_2}{\partial x_1} & \frac{\partial y_2}{\partial x_2} & \cdots & \frac{\partial y_2}{\partial x_n} \\ \vdots & \vdots & \ddots & \vdots \\ \frac{\partial y_m}{\partial x_1} & \frac{\partial y_m}{\partial x_2} & \cdots & \frac{\partial y_m}{\partial x_n} \end{bmatrix} ^ T$$

---

마지막으로 행렬 함수의 미분까지 저희는 정의할 수 있습니다.

> **입력이 행렬**이고 **출력이 스칼라**인 함수의 미분

$$\frac{\partial y}{\partial \bold X} = \begin{bmatrix} \frac{\partial y}{\partial x_{11}} & \frac{\partial y}{\partial x_{12}} & \cdots & \frac{\partial y}{\partial x_{1n}} \\ \frac{\partial y}{\partial x_{21}} & \frac{\partial y}{\partial x_{22}} & \cdots & \frac{\partial y}{\partial x_{2n}} \\ \vdots & \vdots & \ddots & \vdots \\ \frac{\partial y}{\partial x_{m1}} & \frac{\partial y}{\partial x_{m2}} & \cdots & \frac{\partial y}{\partial x_{mn}} \end{bmatrix}$$

> **입력이 스칼라**이고 **출력이 행렬**인 함수의 미분

$$\frac{\partial \bold Y}{\partial x} = \begin{bmatrix} \frac{\partial y_{11}}{\partial x} & \frac{\partial y_{21}}{\partial x} & \cdots & \frac{\partial y_{n1}}{\partial x} \\ \frac{\partial y_{12}}{\partial x} & \frac{\partial y_{22}}{\partial x} & \cdots & \frac{\partial y_{n2}}{\partial x} \\ \vdots & \vdots & \ddots & \vdots \\ \frac{\partial y_{1m}}{\partial x} & \frac{\partial y_{2m}}{\partial x} & \cdots & \frac{\partial y_{nm}}{\partial x} \end{bmatrix}$$

이제 모든 준비를 마쳤고 본격적으로 신경망을 들여다 볼 차례입니다.

기본적인 신경망에서 선형 레이어는 다음과 같은 수식으로 이루어집니다.

$$\bold x^T \bold \it W + \bold b = \bold y$$

이 때, $\bold x$는 입력 벡터, $\bold W$는 가중치 행렬, $\bold b$는 편향 벡터, $\bold y$는 출력 벡터입니다.

---

신경망은 오차함수(Loss Function)을 미분의 연쇄 법칙(Chain rule)에 따라 각 가중치와 입력에 대해 미분해 가면서 기울기를
거슬러 올라가며 계산합니다.

이 때 범위를 좁혀 오차함수의 값을 $E_{total}$이라고 하고 레이어가 한 층이라 할 때 역전파를 보면 가중치 $W$에 대해 미분을 하면
다음과 같이 식이 구성됩니다. (활성화 함수는 계산이 복잡해지니 일단 제외)

$$\frac{\partial E_{total}}{\partial \bold \it W}=\frac{\partial E_{total}}{\partial \bold y}\frac{\partial \bold y}{\partial \bold \it W}$$

이 때 $\frac{\partial \bold y}{\partial \bold W}$는 3차원 텐서라 컴퓨터에 적합한 자료구조가 아닙니다.

따라서 이 식을 수정할 수 있는지 보기위해 전개해보겠습니다.

예를 들어 $\bold y = \bold \it W\bold x(\bold \it W \in \R^{m\times n},\bold x\in \R^n)$일 때
각 원소는 다음과 같습니다.

$$\frac{\partial y_i}{\partial \bold \it W_{jk}} = \begin{cases}x_k &\text{if i = j} \\ 0 &\text{otherwise}\end{cases}$$

즉 이 연산은 i와 j가 같을 때만 원소가 존재합니다.

이것과 같은 것을 생각해보면 나타나는 것이 다음과 같습니다.

$$\frac{\partial E_{total}}{\partial \bold y}\bold x^T$$

이것은  **다음 층에서 건너들어온 기울기**와 **입력**을 **외적**한것과 같습니다.

이 결과는 3차원 텐서의 축소된 표현으로 가능한 이유는 연산의 구조상 특정 차원이 불필요하게 중복되기 때문입니다.

따라서 외적을 이용해서 컴퓨터에서 기울기 계산을 *빠르게 처리*합니다.

이러한 생각을 계속 해보면 앞쪽으로 전달할 입력에 대한 오차함수의 미분도 금방 생각해낼 수 있습니다.

$$\frac{\partial E_{total}}{\partial \bold x} = \frac{\partial E_{total}}{\partial \bold y}\frac{\partial \bold y}{\partial \bold x}=\frac{\partial E_{total}}{\partial \bold y}\bold \it W$$

즉, 오차함수에 대한 입력의 미분은 *가중치 행렬*이 된다는 걸 알 수 있습니다.

## 회고 및 앞으로 할 일

역전파 알고리즘에 대해 수학적으로 엄밀히 정의하고 컴퓨터 알고리즘 적으로도 어떻게 짜야할지 엄밀히 정의했는데
이걸 알기위해 다른 기초적인 개념들이나 그런 걸 배우느라 엄청 힘들었습니다.
자코비안 행렬부터 외적, 편미분 등등 많았지만 그래도 재밌었습니다.
이제 다시 코드를 수정해야겠습니다.

## 프로젝트 깃헙

::github{repo="Ushio-Hayase/Ushionn"}
