---
title: PeekMessage와 GetMessage 비교
published: 2025-02-23
description: "Window API의 GetMessage와 PeekMessage를 비교"
tags: ["WinAPI", "C", "C++"]
category: Knowledge
draft: false
---

# 메시지 큐

윈도우 시스템에서는 메시지 큐가 필요한 스레드가 있으면 메시지 큐를 만듭니다.
각 스레드에서의 메시지 루프는 메시지 큐에서 메시지를 가져오고 적절한 창 프로시저로 전달합니다.
만약 최상위 창이 몇 초 이상 메시지에 응답하지 않는 경우 시스템은 창이 응답하지 않는 것으로 간주하고 z 순서, 위치, 크기 및 시각적 특성이 동일한 고스트 창으로 바꿉니다.
이렇게 하면 사용자가 이동하거나 크기를 조정하거나 애플리케이션을 닫을 수 있습니다.

# 메시지 루프

메시지 루프에서는 `GetMessage`함수와 `DispatchMessage`함수를 사용하여 메시지를 가져오고 전달합니다.
프로그램이 사용자로부터 문자 입력을 가져와야 하는 경우 루프에 `TranslateMessage`함수를 포함합니다.

```cpp
MSG Message;

while (GetMessage(&Message, NULL, 0, 0))
{
    TranslateMessage(&Message);
    DispatchMessage(&Message);
}

```

## GetMessage

GetMessage의 함수 선언은 이렇습니다.

```cpp
BOOL GetMessage(
    LPMSG lpMsg,
    HWND  hWnd,
    UINT  wMsgFilterMin,
    UINT  wMsgFilterMax
);
```

첫번째 매개변수인 lpMsg는 스레드의 메시지 큐에서 메시지 정보를 수신하는 MSG 구조체에 대한 포인터입니다.
두번째 매개변수인 hWnd는 메시지를 검색할 창에 대한 핸들입니다.
이 때 창은 현재 스레드에 속해야합니다.
만약 hWnd가 NULL이라면 GetMessage 함수는 현재 스레드에 속한 모든 창에 대한 메시지 또는 hWnd 값이 NULL인 메시지를 가져옵니다.
hWnd가 -1이라면 GetMessage는 hwnd 값이 NULL인 현재 스레드의 메시지 큐, 즉 PostMessage에서 게시한 스레드 메시지(hWnd 매개 변수가 NULL인 경우) 또는 PostThreadMessage에서 메시지만 검색합니다.
세번째, 네번째 매개변수는 검색할 가장 낮은 메시지 값의 정수 값과 높은 정수값입니다. 
만약 wMsgFilterMin 및 wMsgFilterMax가 모두 0이면 GetMessage는 사용 가능한 모든 메시지를 반환합니다.

:::tip
이 함수는 WM_QUIT 이외의 메시지를 가져오면 0이 아닌 값, WM_QUIT 메시지를 가져오면 0을 반환합니다.
:::

:::warning
추가적으로 이 함수를 메시지 큐에서 WM_PAINT 메시지를 제거하지 않습니다.
:::
## PeekMessage

```cpp
HWND hwnd; 
BOOL fDone; 
MSG msg; 

 
fDone = FALSE; 
while (!fDone) 
{ 
    fDone = DoLengthyOperation();
 
    while (PeekMessage(&msg, hwnd,  0, 0, PM_REMOVE)) 
    { 
        switch(msg.message) 
        { 
            case WM_LBUTTONDOWN: 
                break;
            case WM_RBUTTONDOWN: 
                break;
            case WM_KEYDOWN: 
                fDone = TRUE; 
                break;
        } 
    } 
}
```

PeekMessage 함수 선언은 다음과 같습니다. 

```cpp
BOOL PeekMessageA(
    LPMSG lpMsg,
    HWND  hWnd,
    UINT  wMsgFilterMin,
    UINT  wMsgFilterMax,
    UINT  wRemoveMsg
);
```

GetMessage와 비슷하지만 마지막 매개변수로 메시지를 처리할 방법에 대한 인자가 들어갑니다. 

|값|의미|
|---|---|
|PM_NOREMOVE|PeekMessage 함수를 처리한 후 큐에서 메시지가 제거되지 않습니다|
|PM_REMOVE|PeekMessage 함수를 처리한 후 큐에서 메시지가 제거됩니다|

이외에도 PM_QS_...과 같은 플래그가 있습니다.

이 함수는 메시지를 사용 불가하면 0, 사용 가능하면 0이 아닌 값을 반환합니다.


