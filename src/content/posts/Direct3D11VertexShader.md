---
title: Direct 3D 11의 VertexShader
published: 2025-04-11
description: 'D3D11의 Vertex Shader'
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

각각의 변환은 이미 행렬로서 구현이 되어있어 `CreateVertexShader` 함수를 이용해 생성해주기만 하면 됩니다.

하지만 테셀레이션을 사용할 것이라면 이 셰이더 단계에서 World-View Projection을 수행하면 안됩니다.

왜냐하면 테셀레이션 작업시에 다른 많은 Vertex를 새롭게 생성시키기 때문입니다.