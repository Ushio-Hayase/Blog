---
title: DirectX의 XMVECTOR
published: 2025-03-25
description: "DirectX에서의 FXMVECTOR와 CXMVECTOR의 차이와 FXMVECTOR가 더 효율적인 이유"
tags: ["DirectX"]
category: Knowledge
draft: true
---

# FXMVECTOR와 CXMVECTOR

## 개요

기본적으로 DirectX에서는 벡터를 다룰때 XMVECTOR라는 구조체를 사용합니다.

- 개념 : XMVECTOR는 DirectXMath의 핵심 요소 중 하나로 SIMD(Single Instruction, Multi Data)를 위한 벡터 타입입니다.
XMVECTOR는 주로 4개의 32비트 부동소수점 값(float) 또는 32비트 정수 (int)를 저장하기 위해 설계되었습니다.
- 성능 : XMVECTOR는 SIMD 명령어(x86기준 SSE/SSE2, ARM기준 NEON)을 활용하여 벡터 및 행렬 연산을 한 명령어로 여러 데이터를 동시에 처리해 높은 성능을 얻을 수 있습니다.
- 요구조건 : XMVECTOR는 16바이트 경계에 정렬(16-bytes aligned; 메모리 주소가 16의 배수)되어야 최적의 성능을 발휘합니다.
