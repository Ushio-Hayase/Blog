---
title: CUDA의 메모리 구조와 명령어 실행 단위
published: 2025-05-18
description: 'CUDA에서 논리적으로 실행되는 방식을 알아보고 그것이 NVIDIA GPU에서 어떻게 물리적으로 실행되는지 알아보기'
tags: ["CUDA"]
category: 'Knowledge'
draft: true
lang: 'ko'
---

## CUDA에서의 논리적 실행단위

CUDA C++(.cu, .cuh)에서는 커널 코드는 함수 앞에 `__global__`붙여 CPU에서 GPU에 해당 함수를 실행하게 할 수 있습니다.
이 커널 함수의 반환형은 `void` 형이여야 합니다.
그렇게 만든 커널 함수는 `func<<<grid_size, block_size>>>(args)` 형식으로 호출할 수 있습니다.
만약 CUDA 스트림을 기본 스트림이 아니라 다른 스트림을 사용하고싶다면 `func<<<grid_size, block_size, cuda_stream>>>(args)` 형식으로 호출할 수 있습니다.
현재 NVIDIA GPU의 스레드 블록은 1024개의 스레드를 포함할 수 있습니다.

### 그리드와 블록

CUDA 코드에서 그리드는 전체 디바이스라 할 수 있고 그리드의 크기만큼 블록이 존재합니다.
CUDA에서 블록은 스레드의 집합이라고 할 수 있고 블록의 크기만큼 스레드가 동일한 커널을 공유하며 실행됩니다.
동일한 블록 내에서 스레드들은 *shared memory*를 공유합니다.

## NVIDIA GPU에서의 물리적 실행 단위

위와 같이 생성된 코드는 실제 디바이스에서는 SM(Streaming Multiprocessor)에서 Warp(32개) 단위로 실행됩니다.
로

### SM

SM 내부는 여러가지 부품으로 구성되어있습니다.
메모리 관련으로는 *레지스터 파일*, *공유 메모리*, *상수 캐시*, *텍스처 캐시*, *L1 I-Cache(명령어 캐시)*, *L1 데이터 캐시*, *LD/ST(Load/Store Unit)*, *TMA(Tensor Memory Accelerator)*가 존재합니다.
연산 관련으로는 *FP32 유닛*, *INT32 유닛*, *FP64 유닛*, *텐서 코어*, *SFU(Special Function Unit)*, *RT 코어*, *TMU(Texture Mapping Unit; 텍스처 매핑 유닛)*가 있습니다.
스케줄링 및 제어 관련으로는 *워프 스케줄러*, *디스패치 유닛*, *pc(program counter)*, *Active Mask(활성 마스크)*, *분기 스택(Branch Stack)*, *Register Base Pointer(레지스터 베이스 포인터)*, *Scoreboard(스코어보드)*, *Operand Collector(오퍼랜드 콜렉터)*, *배리어 및 동기화 하드웨어*이 있습니다.

### SM의 메모리 관련 부품 정리

- 레지스터 파일 : SM 내의 가장 대역폭이 높은 SRAM 영역으로, 각 활성 스레드의 변수들을 저장합니다. 만약 블록 내 레지스터 사용량이 SM 내부 레지스터 파일에 존재하는 레지스터 개수를 넘을 경우 데이터가 로컬 메모리(Local Memory)로 넘어가는 **레지스터 스필링(Register Spilling)** 현상이 발생합니다.
  - 레지스터 뱅크 : 레지스터 파일은 레지스터 뱅크 구조로 되어있어 워프내 32개의 스레드에 오퍼랜드를 한사이클만에 공급할 수 있습니다.
  - 레지스터 뱅크 충돌 : 만약 워프 내의 모든 스레드가 같은 뱅크에 존재하는 레지스터에 접근한다면 각 뱅크는 한사이클에 1개만 오퍼랜드를 줄 수 있어 지연이 발생합니다. 하지만 일반적으로 컴파일러가 이를 피하여 레지스터를 할당하기에 대부분의 상황에서는 신경쓸 필요가 없습니다.
- 공유 메모리 : 블록 내
