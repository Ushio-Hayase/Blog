---
title: DirectX의 InputAssembler
published: 2025-04-08
description: '다이렉트X 3D의 그래픽 파이프라인 단계 중 하나인 IA(Input-Assembler)에 대해 알아봅니다'
image: ''
tags: ["DirectX"]
category: 'Knowledge'
draft: true
lang: 'ko'
---

## 개요

Input-Assembler는 그래픽 파이프라인에서 첫 번째 단계를 담당하고 있는 단계입니다.

IA(Input-Assembler)는 primitive data(점, 선, 삼각형)를 유저가 채운 버퍼에서 읽고
다른 파이프라인 단계에서 쓸 수 있도록 모읍니다.

이 모은 데이터는 다음 그래픽 파이프라인 단계인 GS(Geometry Shader)에 사용될 수 있도록
인접한 점들끼리 모읍니다.


