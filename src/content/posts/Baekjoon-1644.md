---
title: Baekjoon-1644
published: 2025-04-19
description: '백준 1644번 C++ 풀이'
tags: ["C++", "P.S.", "Baekjoon"]
category: 'P.S.'
draft: false 
lang: 'ko'
---

## 문제

하나 이상의 연속된 소수의 합으로 나타낼 수 있는 자연수들이 있다. 몇 가지 자연수의 예를 들어 보면 다음과 같다.

3 : 3 (한 가지)
41 : 2+3+5+7+11+13 = 11+13+17 = 41 (세 가지)
53 : 5+7+11+13+17 = 53 (두 가지)
하지만 연속된 소수의 합으로 나타낼 수 없는 자연수들도 있는데, 20이 그 예이다. 7+13을 계산하면 20이 되기는 하나 7과 13이 연속이 아니기에 적합한 표현이 아니다. 또한 한 소수는 반드시 한 번만 덧셈에 사용될 수 있기 때문에, 3+5+5+7과 같은 표현도 적합하지 않다.

자연수가 주어졌을 때, 이 자연수를 연속된 소수의 합으로 나타낼 수 있는 경우의 수를 구하는 프로그램을 작성하시오.

## 입력

첫째 줄에 자연수 N이 주어진다. (1 ≤ N ≤ 4,000,000)

## 출력

첫째 줄에 자연수 N을 연속된 소수의 합으로 나타낼 수 있는 경우의 수를 출력한다.

## 풀이과정

이 문제는 연속된 합이라는 단어를 통해 부분합과 투 포인터를 이용하면 $O(N)$ 의 시간으로 합 구하는 건 쉽게 할 수 있겠다는 생각이 들었습니다.

하지만 정작 헷갈렸던건 소수를 구하는 부분이였습니다.

이 문제는 N의 범위가 4,000,000까지 가능해 에라토스테네스의 체를 이용한 알고리즘을 제대로 구현하여 시간을 최대한 아끼는게 힘들었습니다.

처음에는 2부터 N까지 모두 나눠보며 에라토스테네스의 체를 구현했지만 당연히 실패하였습니다.

이 후 검사는 $\sqrt{N}$까지만 해도 된다는 걸 상기해냈고 검사할 때 i번째 일 때 $i * i$ 미만의 수들은 이미 검사가 되있을 것이라는 것을 깨달아 시간을 더 줄이는 코드를 작성했습니다.

그리하여 최종 코드는 다음과 같습니다.

## 코드

```cpp
#include <cmath>
#include <iostream>
#include <vector>

using namespace std;

int N;

int main()
{
    cin >> N;

    vector<int> isPrime(N + 1, true);
    vector<int> primeNumbers(1);
    vector<int> partialSum(2);
    primeNumbers[0] = 2;
    partialSum[0] = 0;
    partialSum[1] = 2;

    if (N == 1)
    {
        cout << 0;
        return 0;
    }
    else if (N == 2)
    {
        cout << 1;
        return 0;
    }

    for (int i = 2; i <= sqrt(N) + 1; ++i)
    {
        if (isPrime[i])
        {
            for (int j = i * i; j <= N; j += i)
            {
                isPrime[j] = false;
            }
        }
    }

    for (int i = 3; i <= N; ++i)
    {
        if (isPrime[i]) partialSum.emplace_back(partialSum.back() + i);
    }

    const int len = partialSum.size();
    int left = 0, right = 1;

    int result = 0;

    while (right < len)
    {
        const int sum = partialSum[right] - partialSum[left];
        if (sum == N)
        {
            result++;
            left++;
        }
        else if (sum > N)
            left++;
        else if (sum < N)
            right++;
    }

    cout << result;
}
```
