---
title: "DirectX3D 초기화"
published: 2025-03-29
description: "DirectX3D를 초기화하고 화면에 출력할 때까지의 필요한 요소를 찾고 탐구해봅니다."
tags: ["DirectX"]
category: Knowledge
draft: false
---

## 개요

DirectX3D는 프로그래머와 그래픽 하드웨어 사이에서 추상화를 담당하는 중재자라고 할 수 있습니다.
프로그래머는 추상화된 계층이 있어 하드웨어의 명령어를 알 필요까지 없게 하고 하드웨어 설계자도 DirectX의 요구 사항에 맞춰
설계하여 사용하기 쉽게 만들 수 있습니다.

여기서는 DirectX3D의 초기화 과정을 순차적으로 따라가며 각각의 요소가 무엇을 의미하는지 차근차근 알아가 볼 것입니다.

DirectX3D의 화면에 초기화할 때까지 과정을 일반적으로 다음과 같습니다.

1. 디바이스 인터페이스와 디바이스 컨텍스트 인터페이스 생성성
2. 스왑체인 생성
3. 렌더 타겟 뷰 생성
4. Depth/Stencil 뷰 생성
5. 뷰들을 출력 병합기 (Output Merger)에 묶기
6. 뷰포트 설정

## DirectX의 텍스처

DirectX에서 1차원 텍스처는 1차원 배열과 비슷하고, 2차원 텍스처는 2차원 배열과 비슷하고,
3차원 텍스처는 3차원 배열과 비슷합니다.
이 텍스처의 원소들의 형식(Format)은 DXGI_FORMAT 열거형의 한 멤버에 반드시 해당해야 합니다.

일반적으로 텍스처는 이미지 자료를 담지만, 깊이(Depth) 정보와 같은 다른 자료를 담는 것도 가능합니다.

GPU는 텍스처에 대해 필터링이라든가 다중 표본 추출과 같은 연산을 수행할 수 있습니다.

## Device Interface && Device Context Interface

이 ID3D11Device(Device Interface)와 ID3D11DeviceContext(Device Context Interface)는 각각
기능 지원 점검과 자원 할당 / 렌더 대상 설정 및 그래픽 파이프라인에 묶기, GPU가 수행할 렌더링 명령 지시와 같은 역할을 수행하는 객체입니다.

ID3D11Device와 ID3D11DeviceContext는 D3D11CreateDevice [^1]함수를 통해 생성합니다.

[^1]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-d3d11createdevice)

## 스왑체인(SwapChain;교환 사슬)

DirectX는 화면을 2개의 버퍼를 이용하여 그립니다.

정확히는 후면 버퍼와 전면 버퍼가 있고 한 프레임의 장면 전체를 후면 버퍼에 그린 후 후면 버퍼와 전면 버퍼의
역할을 맞바꾸고 맞바뀐뒤 전면 버퍼를 화면에 표시하며(Presenting; 제시) 바꾸기 전 전면버퍼는 후면 버퍼가 되어 장면을 그리는 걸 반복합니다.

이 때 전면 버퍼와 후면 버퍼의 맞바꾸는 것은 포인터만 맞바꾸는 것이기에 효율적인 연산입니다.

이 후면 버퍼의 개수에 따라 1개일땐 이중 버퍼링, 2개일 땐 삼중 버퍼링이라고 부릅니다.

이 버퍼는 크기 변경을 위한 메서드(IDXGISwapChain::ResizeBuffers)와 버퍼의 제시를 위한 메서드(IDXGISwapChain::Present)를 제공합니다. [^2]

스왑체인은 IDXGIFactory::CreateSwapChain [^3]메서드를 사용해서 생성합니다.

[^2]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/dxgi/nn-dxgi-idxgiswapchain)
[^3]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/dxgi/nf-dxgi-idxgifactory-createswapchain)

## 렌더 대상 뷰(Render Target View;렌더 타겟 뷰)

DirectX에서는 자원이 있을 때 텍스처 자원이 파이프라인의 단계에 직접 묶이진 않습니다.

대신에 그 텍스처의 자원 뷰(Resource View)를 생성해서 사용합니다.

그 자원 뷰가 있으므로 인해서 런타입과 드라이버가 유효성 점검과 매핑을 뷰 생성 시점에서 수행할 수 있기 때문에 묶는 시점에서
형식 점검이 최소화됩니다.

자원 뷰를 사용할 때는 Direct3D에게 자원의 사용 방식(파이프라인의 어느 단계에 묶을건지)을 알려줘야하고, 생성 시점에서 무형식을 지정한 자원 형식의 구체적인 형식을 결정해야합니다.

무형식 자원은 텍스처 원소를 한 파이프라인 단계에서는 부동소수점 값으로서 사용하고 다른 단계에서는 정수로서 사용가능함을
의미합니다.

## 깊이-스텐실(Depth-Stencil) 버퍼와 뷰 생성

깊이 버퍼는 각 픽셀의 깊이 정보를 담습니다.

Direct3D는 이 깊이 버퍼를 사용해서 어떤 물체가 다른 물체 앞에 있는지 판별하기 위해 필요합니다.

깊이 버퍼링 알고리즘은 렌더링되는 각 픽셀의 깊이 값을 계산해서 깊이 판정을 수행합니다.

이 수행된 픽셀은 관찰자에게 가장 가까운 픽셀이 승자가 되어 후면 버퍼에 기록됩니다.

깊이 버퍼같은 2차원 텍스처를 생성하기 위해선 D3D11_TEXTURE_DESC [^4]구조체를 채우고 ID3D11Device::CreateTexture2D [^5]메서드를 호출해야합니다.

[^4]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/ns-d3d11-d3d11_texture2d_desc)
[^5]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11device-createtexture2d)

## 뷰들을 출력 병합기(Output Merger)에 묶고 뷰포트 설정

이제 뷰들을 다 생성했으니 자원들이 파이프라인의 렌더 대상과 깊이-스텐실 버퍼로 작용하도록 출력 병합기에 묶어야 합니다.

이 작업은 디바이스 컨텍스트의 OMSetRenderTargets [^6]메서드를 사용해서 묶습니다.

마지막으로 장면의 일부인 뷰포트를 디바이스 컨텍스트의 RSSetViewports [^7]메서드를 사용해서 Direct3D에게 알려줍니다.

[^6]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-omsetrendertargets)
[^7]: [MS_Learn_Docs](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-rssetviewports)

## 참조

> DirectX 11을 이용한 3D 게임 프로그래밍 입문
