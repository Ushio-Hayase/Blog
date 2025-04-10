---
title: DirectX의 InputAssembler
published: 2025-04-08
description: '다이렉트 3D의 그래픽 파이프라인 단계 중 하나인 IA(Input-Assembler)에 대해 알아봅니다'
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

데이터의 Adjacency는 GS에만 적용되는데 이 것을 적용하면 각 삼각형마다 3개의 Adjancency 데이터가
추가로 데이터에 포함됩니다.

만약 면의 끝 쪽이라거나의 이유로 정점이 없더라도 데이터를 넣을 땐 더미 정점이라도 넣어야 합니다.

이 IA 단계는 Input 버퍼를 만들고 Input-Layout 객체를 만들고 IA 단계에 묶는 방식으로 진행됩니다.

## Input Buffers

이 인풋 버퍼에는 정점 버퍼와 인덱스(색인) 버퍼, 2가지 종류가 있습니다.

정점 버퍼는 IA 단계에 정점 데이터를 공급해주는 버퍼로 필수적인 버퍼입니다.

인덱스 버퍼는 정점 버퍼로부터의 색인을 정점에 공급해주는 버퍼입니다.

이 정점 버퍼들은 여러개 만들 수 있습니다.

정점 버퍼는 프로그래머가 구조체를 정의하고 그 구조체로 만든 배열과 구조체에 대한 정보가 든
Direct 3D가 지원하는 구조체(D3D11_BUFFER_DESC [^1])를 이용해 생성합니다.

생성할 때는 디바이스 객체의 CreateBuffer 메서드를 사용해서 생성합니다.

[^1]: [D3D11_BUFFER_DESC_MS_Learn](https://learn.microsoft.com/ko-kr/windows/win32/api/d3d11/ns-d3d11-d3d11_buffer_desc)

## Input-Layout Object

여기서는 앞서 생성한 버퍼에 대한 정보를 추가적으로 Direct 3D에 제공합니다.

D3D11_INPUT_ELEMENT_DESC [^2] 배열로 생성해서 디바이스 객체의 CreateInputLayout 메서드로 Direct 3D에 등록합니다.

이 구조체의 각 요소는 각 정점의 데이터를 묘사합니다.

이 인풋 레이아웃 객체는 셰이더의 인풋 시그니처에 맞기만하면 여러번 사용해도 됩니다.

[^2]: [D3D11_INPUT_ELEMENT_DESC](https://learn.microsoft.com/ko-kr/windows/win32/api/d3d11/ns-d3d11-d3d11_input_element_desc)

## 단계에 바인딩

생성한 오브젝트들은 각각 디바이스 컨텍스트 객체의 IASetVertexBuffers, IASetIndexBuffer, IASetInputLayout 메서드를
통해 단계에 바인딩합니다.
