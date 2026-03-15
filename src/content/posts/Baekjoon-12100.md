---
title: Baekjoon-12100
published: 2025-04-16
description: '백준 12100번 문제 C++ 풀이과정'
tags: ["C++", "P.S.", "Baekjoon"]
category: 'P.S.'
draft: false 
lang: 'ko'
---

## 문제

[링크](https://www.acmicpc.net/problem/12100)

문제가 길어 링크를 첨부했습니다.

## 입력

첫째 줄에 보드의 크기 N (1 ≤ N ≤ 20)이 주어진다. 둘째 줄부터 N개의 줄에는 게임판의 초기 상태가 주어진다. 0은 빈 칸을 나타내며, 이외의 값은 모두 블록을 나타낸다. 블록에 쓰여 있는 수는 2보다 크거나 같고, 1024보다 작거나 같은 2의 제곱꼴이다. 블록은 적어도 하나 주어진다.

## 출력

최대 5번 이동시켜서 얻을 수 있는 가장 큰 블록을 출력한다.

## 풀이

먼저 이 문제는 블럭들을 4방향으로 움직이고 합치는 것이 중점입니다.

이 것을 위해 매 단계마다 보드 전체를 탐색하며 블럭이 있으면 해당하는 방향으로 움직이고 합치기로 생각했습니다.

보드 크기는 최대 20이고 5번까지 움직일 수 있다 조건이 주어져 있어 최악의 경우 반복횟수는 $20*20*4^5 = 409,600$ 으로
제가 생각하는 최대 반복횟수인 10억번에 미치지 않아 충분히 제한시간 안에 풀릴 것이라 생각했습니다.

이 문제의 충돌은 블록이 있는 칸에서 가장 마지막으로 탐색한 칸을 저장한 다음 해당하는 칸의 숫자가 현위치 블럭의 숫자와
같으면 충돌하게 구현하였습니다.

조건에서 충돌은 블럭 당 1번이라 그랬는데 이 충돌을 매 행이나 열마다 충돌 여부 배열을 만들어 이번 회차에 충돌했는지
여부를 기록하고 사용해 이 조건을 구현하였습니다.

충돌이 아닌 이동의 경우는 아까 사용했던 블럭이 있는 마지막 탐색 칸의 한 칸 앞으로 바로 이동시키는 것으로 구현했습니다.

이렇게 이동 및 충돌 구현은 쉬웠지만 정작 헤맸던 부분은 객체 수명 관리였습니다.

처음 시도하고 계속 오류가 났는데 시도하고 코드를 탐색하다 보니 아래와 같이 4방향으로 탐색할 때 같은 행렬을
사용하는 실수를 저질렀습니다.

```cpp
    vector<vector<int>> arr{q.front().second};
        q.pop();

        for (int i = 0; i < 4; ++i)
        {
            moveArr(arr, dx[i], dy[i]);
        ...
```

이 것을 해결하기 위해 지역변수로 선언하고 참조로 넘겼더니 객체가 사라지는 문제가 있었고 결국엔 객체를 깊은 복사 생성자로
매 분기마다 복사하는 것으로 최종적으로 구현했습니다.

## 코드

```cpp
#include <iostream>
#include <queue>

using namespace std;

int N;

int dx[4]{1, -1, 0, 0};
int dy[4]{0, 0, 1, -1};

void moveArr(vector<vector<int>>& arr, int deltaX, int deltaY)
{
    if (deltaX > 0)
    {
        for (int j = 0; j < N; ++j)
        {
            int blockCol = N;  // 블록이 있는 칸 중 열이 가장 작은 수
            vector<bool> notMerge(N);
            for (int i = N - 1; i >= 0; --i)
            {
                if (arr[j][i] == 0) continue;

                // 현위치 블록과 블록이 도착할 칸 다음 칸 블록이 같은 수라면
                if (blockCol < N && arr[j][i] == arr[j][blockCol] &&
                    !notMerge[blockCol])
                {
                    arr[j][blockCol] *= 2;
                    arr[j][i] = 0;
                    notMerge[blockCol] = true;
                }
                else
                {
                    arr[j][blockCol - 1] = arr[j][i];
                    if (blockCol - 1 != i) arr[j][i] = 0;
                    blockCol -= 1;  // 블록이 있는 칸 갱신
                }
            }
        }
    }
    else if (deltaX < 0)
    {
        for (int j = 0; j < N; ++j)
        {
            int blockCol = -1;  // 블록이 있는 칸 중 열이 가장 큰 수
            vector<bool> notMerge(N);
            for (int i = 0; i < N; ++i)
            {
                if (arr[j][i] == 0) continue;

                // 현위치 블록과 블록이 도착할 칸 다음 칸 블록이 같은 수라면
                if (blockCol > -1 && arr[j][i] == arr[j][blockCol] &&
                    !notMerge[blockCol])
                {
                    arr[j][blockCol] *= 2;
                    arr[j][i] = 0;
                    notMerge[blockCol] = true;
                }
                else
                {
                    arr[j][blockCol + 1] = arr[j][i];
                    if (blockCol + 1 != i) arr[j][i] = 0;
                    blockCol += 1;  // 블록이 있는 칸 갱신
                }
            }
        }
    }
    else if (deltaY > 0)
    {
        for (int j = 0; j < N; ++j)
        {
            int blockRow = N;  // 블록이 있는 칸 중 행이 가장 작은 수
            vector<bool> notMerge(N);
            for (int i = N - 1; i >= 0; --i)
            {
                if (arr[i][j] == 0) continue;

                // 현위치 블록과 블록이 도착할 칸 다음 칸 블록이 같은 수라면
                if (blockRow < N && arr[i][j] == arr[blockRow][j] &&
                    !notMerge[blockRow])
                {
                    arr[blockRow][j] *= 2;
                    arr[i][j] = 0;
                    notMerge[blockRow] = true;
                }
                else
                {
                    arr[blockRow - 1][j] = arr[i][j];
                    if (blockRow - 1 != i) arr[i][j] = 0;
                    blockRow -= 1;  // 블록이 있는 칸 갱신
                }
            }
        }
    }
    else if (deltaY < 0)
    {
        for (int j = 0; j < N; ++j)
        {
            int blockRow = -1;
            vector<bool> notMerge(N);
            for (int i = 0; i < N; ++i)
            {
                if (arr[i][j] == 0) continue;
                if (blockRow > -1 && arr[i][j] == arr[blockRow][j] &&
                    !notMerge[blockRow])
                {
                    arr[blockRow][j] *= 2;
                    arr[i][j] = 0;
                    notMerge[blockRow] = true;
                }
                else
                {
                    arr[blockRow + 1][j] = arr[i][j];
                    if (blockRow + 1 != i) arr[i][j] = 0;
                    blockRow += 1;  // 블록이 있는 칸 갱신
                }
            }
        }
    }
}

int bfs(const vector<vector<int>>& init)
{
    queue<pair<int, vector<vector<int>>>> q;

    q.push({0, init});

    int result = 0;

    while (!q.empty())
    {
        const int level = q.front().first;

        for (int i = 0; i < 4; ++i)
        {
            vector<vector<int>> arr(q.front().second);
            moveArr(arr, dx[i], dy[i]);
            if (level == 4)
            {
                int tmp = 0;
                for (int j = 0; j < N; ++j)
                    for (int k = 0; k < N; ++k)
                        result = arr[j][k] > result ? arr[j][k] : result;
            }
            else
            {
                q.push({level + 1, arr});
            }
        }
        q.pop();
    }

    return result;
}

int main()
{
    vector<vector<int>> arr;
    cin >> N;

    arr.resize(N);

    for (int i = 0; i < N; ++i)
    {
        arr[i].resize(N);
        for (int j = 0; j < N; ++j)
        {
            cin >> arr[i][j];
        }
    }

    cout << bfs(arr);
}
```
