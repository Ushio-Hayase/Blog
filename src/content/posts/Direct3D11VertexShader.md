---
title: Direct 3D 11의 VertexShader
published: 2025-04-11
description: 'D3D11의 Vertex Shader의 구조'
image: ''
tags: ["DirectX"]
category: 'Knowledge'
draft: false 
lang: 'ko'
---

## 개요

Vertex Shader는 하는 일이 매우 심플합니다.

각자 자신만의 좌표계에 있는 정점 데이터들을 한 좌표계로 모으고
그걸 카메라의 좌표계로 변환하고 카메라 시야 안에 있는 것들만 사용하며
원근법을 적용합니다.

각각의 변환은 이미 행렬로서 구현이 되어있어 `CreateVertexShader` 함수를 이용해 생성해주기만 하면 된다.
