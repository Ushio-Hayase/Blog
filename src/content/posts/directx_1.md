---
title: DirectX의 XMVECTOR
published: 2025-03-28
description: "DirectX에서의 FXMVECTOR와 CXMVECTOR의 차이와 때에 따라 사용하는 법"
tags: ["DirectX", "C++"]
category: Knowledge
draft: false
---


## 개요

기본적으로 DirectX에서는 벡터를 다룰때 XMVECTOR라는 구조체를 사용합니다.

- 개념 : XMVECTOR는 DirectXMath의 핵심 요소 중 하나로 SIMD(Single Instruction, Multi Data)를 위한 벡터 타입입니다.
XMVECTOR는 주로 4개의 32비트 부동소수점 값(float) 또는 32비트 정수 (int)를 저장하기 위해 설계되었습니다.
- 성능 : XMVECTOR는 SIMD 명령어(x86기준 SSE/SSE2/AVX...etc, ARM기준 NEON)을 활용하여 벡터 및 행렬 연산을 한 명령어로 여러 데이터를 동시에 처리해 높은 성능을 얻을 수 있습니다.
- 요구조건 : XMVECTOR는 16바이트 경계에 정렬(16-bytes aligned; 메모리 주소가 16의 배수)되어야 최적의 성능을 발휘합니다.

여기서 XMVECTOR가 각 아키텍처마다 어떻게 변환되는지 알아보고 각 대상의 차이점 및 공통점을 알아볼 예정입니다.

## XMVECTOR

XMVECTOR는 SIMD 연산을 위한 기본적인 타입입니다.
이것 자체는 플랫폼 별로 있는 기본 SIMD 벡터 타입에 대한 `typedef` 또는 `using`을 이용한 별칭입니다.

| XMVECTOR 자체의 정의     | 기본 SIMD 벡터 타입 |
| ------------------------ | ------------------- |
| x86(32bit,SSE/SSE2 지원) | __m128              |
| x64                      | __m128              |
| ARM(32bit, NEON 지원)    | float32x4_t         |
| ARM(64bit)               | float32x4_t         |

XMVECTOR는 크게 FXMVECTOR & GXMVECTOR & HXMVECTOR와 CXMVECTOR로 플랫폼 별로 최적의 호출 규약에 맞추어 추상화될 수 있습니다.

## FXMVECTOR & GXMVECTOR & HXMVECTOR와 CXMVECTOR

FXMVECTOR & GXMVECTOR & HXMVECTOR와 CXMVECTOR는 모두 XMVECTOR를 효율적으로 사용하기 위한 타입 별칭입니다.

아래 표는 각각의 형태가 어떤 타입의 별칭인지가 나와있는 표입니다.

| 아키텍처 | FXMVECTOR 정의    | GXMVECTOR 정의    | HXMVECTOR 정의    | CXMVECTOR 정의    | 비고                                                           |
| -------- | ----------------- | ----------------- | ----------------- | ----------------- | -------------------------------------------------------------- |
| x86      | `const XMVECTOR&` | `const XMVECTOR&` | `const XMVECTOR&` | `const XMVECTOR&` | 레지스터로 전달이 비효율적                                     |
| x64      | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR&` | __vectorcall 사용시 HXMVECTOR로 전달하는 인자의 개수 확장 가능 |
| ARM      | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR&` | NEON 레지스터 활용                                             |
| ARM64    | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR`  | `const XMVECTOR&` | NEON 레지스터 활용                                             |

이 표에서 `const XMVECTOR`는 규약마다 다르지만 일반적으로는 레지스터로 로드하여 접근해 메모리 접근보다 빠르게 실행할 수 있습니다.

### x64 아키텍처에서 전달 방식

일반적으로 벡터를 이용하는 함수는 처음에 오는 3개의 인자를 FXMVECTOR의 형식으로 전달하라고 합니다.
정확히는 처음에 오는 XMVECTOR 형식의 인자 3개까지를 각각 FXMVECTOR -> GXMVECTOR -> HXMVECTOR 형식으로 전달해야 효율적입니다.
이것은 Microsoft DirectXMath 라이브러리의 권장사항인데 주로 Microsoft의 x64 호출 규약과 관련이 깊습니다.

이 x64 호출 규약에는 `__fastcall`과 `__vectorcall`이 포함되는데 일반적으로는 `__fastcall` 규약이 사용됩니다.
이 `__fastcall` 규약은 첫 인자 3개까지를 FXMVECTOR 형태로 넘기라고하는데 그것은 XMM0-XMM3까지의 레지스터에 저장됩니다.

하지만 프로그래머가 `__vectorcall` 규약을 함수 이름 앞에 명시해서 바꿀 경우 벡터 인자를 최대 6개까지 레지스터로 전달할 수 있습니다. (XMM0-XMM5)

이 규약들을 정할 땐 XMVECTOR 뿐만 아니라 다른 인자들도 레지스터를 차지한다는 점을 생각하며 잘 선택해야 합니다.

## 더 병렬 처리 성능을 높이는 법

여기서 더 성능을 내기위해 AVX 명령어 셋을 이용할 수 있습니다.
사용하는 방법은 컴파일러가 AVX 명령어를 사용하도록 인자를 줘서 컴파일 하면 됩니다. (MSVC 경우 /arch:AVX, /arch:AVX2, /arch:AVX512)

이렇게 하면 컴파일러가 더 효율적일 수도 있는 명령어로 바꿔주어 더 성능이 향상될 수도 있습니다.

## 결론

이렇게 XMVECTOR가 어떻게 나뉘는지 컴파일시 어떤 명령어와 규약이 어떤 형태로 작동하는지 살펴봤습니다.

이러한 규약과 명령어 셋, 타입들을 프로그래머들은 잘 선택해 성능이 잘 나오도록 프로그래밍 해야할 것입니다.

## 참조

[MS_Learn_DirectXMath](https://learn.microsoft.com/en-us/windows/win32/dxmath/pg-xnamath-internals#parameter-passing)  

[__vectorcall](https://learn.microsoft.com/en-us/cpp/cpp/vectorcall)  

[x64_호출규약](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention)
