---
title: 백준 11866번 C++ 풀이
published: 2025-03-06
description: "백준 11866번 C++ 풀이"
tags: ["P.S.", "C++", "Baekjoon"]
category: P.S.
---

# 백준 11866번 C++ 풀이

## 문제 

> 요세푸스 문제는 다음과 같다.
> 
> 1번부터 N번까지 N명의 사람이 원을 이루면서 앉아있고, 양의 정수 K(≤ N)가 주어진다. 이제 순서대로 K번째 사람을 제거한다. 한 사람이 제거되면 남은 사람들로 이루어진 원을 따라 이 과정을 계속해 나간다. 이 과정은 N명의 사람이 모두 제거될 때까지 계속된다. 원에서 사람들이 제거되는 순서를 (N, K)-요세푸스 순열이라고 한다. 예를 들어 (7, 3)-요세푸스 순열은 <3, 6, 2, 7, 5, 1, 4>이다.
> 
> N과 K가 주어지면 (N, K)-요세푸스 순열을 구하는 프로그램을 작성하시오.

## 입력

> 첫째 줄에 N과 K가 빈 칸을 사이에 두고 순서대로 주어진다. (1 ≤ K ≤ N ≤ 1,000)

## 출력

> 예제와 같이 요세푸스 순열을 출력한다.

## 예제 입력

> `입력:` 7 3  
> `출력:` <3, 6, 2, 7, 5, 1, 4>

## 아이디어

1. 리스트로 원소를 저장한다. 
2. 원형 큐처럼 끝과 끝을 이어 K번째마다 요소를 빼내고 출력한다.

## 1차 시도

```cpp
#include <cmath>
#include <iostream>
#include <list>

using namespace std;

int N, K;
std::list<int> arr;

int main()
{
    cin >> N >> K;
    for (int i = 0; i < N; ++i)
    {
        arr.emplace_back(i + 1);
    }

    cout << "<";

    int i = 1;
    auto pr = arr.begin();

    while (!arr.empty())
    {
        if (i % K == 0)
        {
            cout << (arr.size() == N ? "" : ", ") << *pr;
            pr = arr.erase(pr);
        }
        else
            pr++;
        i++;

        if (pr == arr.end())
        {
            pr = arr.begin();
        }
    }
    cout << ">";
}
```


:::note[결과]  
맞았습니다!!


[깃헙 링크](https://github.com/Ushio-Hayase/Baekjoon/tree/main/%EB%B0%B1%EC%A4%80/Silver/11866.%E2%80%85%EC%9A%94%EC%84%B8%ED%91%B8%EC%8A%A4%E2%80%85%EB%AC%B8%EC%A0%9C%E2%80%850)