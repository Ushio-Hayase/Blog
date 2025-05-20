---
title: 0-1 bfs [시간복잡도가 O(E+V)인 다익스트라]
published: 2025-05-19
description: '가중치의 0 또는 1일 때 시간 복잡도가 O(E+V)인 다익스트라 알고리즘의 변형 알고리즘'
tags: ["Algorithm"]
category: 'Knowledge'
draft: false 
lang: 'ko'
---

## 개요

다익스트라 알고리즘은 간선의 개수가 $E$이고 정점의 개수가 $V$일 때 시간복잡도가 $O((E+V)\log V)$입니다.

하지만 가중치가 0과 1(정확히는 양의 정수)만 존재할 시 이것을 더 최적화 할 수 있는 방법이 있습니다.

그것이 바로 0-1 bfs입니다.

어떻게 이게 가능하냐면 일반적인 다익스트라 알고리즘은 우선순위 큐를 사용하지만 이 알고리즘은 덱 자료구조를 이용하여 우선순위 큐를 대체하기 때문입니다.

## 동작 순서

1. 덱(deque)을 사용합니다.
2. 덱에서 더 이상 꺼낼 노드가 없을 때까지 다음 과정을 반복합니다.
   1. 덱의 front에서 현재 노드를 꺼냅니다.
   2. 연결된 인접 노드를 살펴봅니다.
   3. 현재 노드까지의 거리 + 그 노드까지의 가중치 < 그 노드까지의 거리면 거리를 갱신합니다.
      1. 현재 노드에서 다음 노드로 가는 간선의 가중치가 0이면, 다음 노드를 덱의 **앞**(front)에 넣습니다.
      2. 가중치가 1 (또는 다른 양의 상수 W)이면, 다음 노드를 덱의 **뒤**(back)에 넣습니다.

## 동작 원리

가중치가 0인 간선은 "비용 없이" 이동하는 것과 같습니다.

따라서 이런 간선들은 최대한 먼저 탐색해도 현재까지의 누적 거리가 늘어나지 않습니다.

덱의 앞에 넣음으로써 이들을 우선적으로 처리합니다.

가중치가 1인 간선은 비용을 1 증가시킵니다.

따라서 이들은 0인 간선들을 모두 처리한 후에 꺼냅니다.

결과적으로, 덱에서 노드를 꺼낼 때 항상 현재까지의 누적 거리가 가장 작은 (또는 같은 거리 중에서는 먼저 탐색된) 노드를 꺼내게 되어 다익스트라 알고리즘과 유사한 효과를 냅니다.

## 시간복잡도 분석

아래 코드를 보면 알겠지만 정점($V$)동안 최악의 경우 모든 간선($E$)를 시간 복잡도 $O(1)$로 접근해서 탐색하므로 최종 시간복잡도는 다음과 같습니다.

$$O(V+E)$$

## 코드

```cpp
// C++ program to implement single source
// shortest path for a Binary Graph
#include<bits/stdc++.h>
using namespace std;

/* no.of vertices */
#define V 9

// a structure to represent edges
struct node
{
    // two variable one denote the node
    // and other the weight
    int to, weight;
};

// vector to store edges
vector <node> edges[V];

// Prints shortest distance from given source to
// every other vertex
void zeroOneBFS(int src)
{
    // Initialize distances from given source
    int dist[V];
    for (int i=0; i<V; i++)
        dist[i] = INT_MAX;

    // double ende queue to do BFS.
    deque <int> Q;
    dist[src] = 0;
    Q.push_back(src);

    while (!Q.empty())
    {
        int v = Q.front();
        Q.pop_front();

        for (int i=0; i<edges[v].size(); i++)
        {
            // checking for the optimal distance
            if (dist[edges[v][i].to] > dist[v] + edges[v][i].weight)
            {
                dist[edges[v][i].to] = dist[v] + edges[v][i].weight;

                // Put 0 weight edges to front and 1 weight
                // edges to back so that vertices are processed
                // in increasing order of weights.
                if (edges[v][i].weight == 0)
                    Q.push_front(edges[v][i].to);
                else
                    Q.push_back(edges[v][i].to);
            }
        }
    }

    // printing the shortest distances
    for (int i=0; i<V; i++)
        cout << dist[i] << " ";
}

void addEdge(int u, int v, int wt)
{
   edges[u].push_back({v, wt});
   edges[v].push_back({u, wt});
}

// Driver function
int main()
{
    addEdge(0, 1, 0);
    addEdge(0, 7, 1);
    addEdge(1, 7, 1);
    addEdge(1, 2, 1);
    addEdge(2, 3, 0);
    addEdge(2, 5, 0);
    addEdge(2, 8, 1);
    addEdge(3, 4, 1);
    addEdge(3, 5, 1);
    addEdge(4, 5, 1);
    addEdge(5, 6, 1);
    addEdge(6, 7, 1);
    addEdge(7, 8, 1);
    int src = 0;//source node
    zeroOneBFS(src);
    return 0;
}
```

- [code from GeekforGeeks](https://www.geeksforgeeks.org/0-1-bfs-shortest-path-binary-graph/)

## 백준 예시 문제

- [13649번 숨바꼭질 3](https://www.acmicpc.net/problem/13549)
