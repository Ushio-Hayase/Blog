---
title: 윈도우 메시지 루프 논블로킹 메시지 처리
published: 2025-02-23
description: "Window API의 GetMessage와 PeekMessage를 애니메이션 동작을 이용하여 비교"
tags: ["WinAPI", "C", "C++"]
category: Project
draft: false
---


# 목표
Windows 메시지 루프는 게임 개발에서 매우 중요한 개념입니다. 
이 글에서는 GetMessage()와 PeekMessage()의 차이를 비교하고, 이를 활용해 간단한 애니메이션을 구현할 것입니다.

# 비교할 코드 스니펫

- GetMessage를 활용한 코드드
```cpp
while (true)
{
    while (GetMessage(&Message, NULL, 0, 0))
    {
        TranslateMessage(&Message);
        DispatchMessage(&Message);
    }
    UpdateGameLogic(hWnd);
}
```

- PeekMessage를 활용한 코드드
```cpp
while (true)
{
    while (PeekMessage(&Message, NULL, 0, 0, PM_REMOVE))
    {
        TranslateMessage(&Message);
        DispatchMessage(&Message);
    }
    UpdateGameLogic(hWnd);
}
```

# 실행 결과

![GetMessage](./Blocking.png)

GetMessage를 활용한 코드는 첫 위치에서 더 이상 움직이지 않는걸 볼 수 있습니다.

![PeekMessage1](./NonblockingFront.png)
![PeekMessage2](./NonblockingBack.png)

반면, PeekMessage를 활용한 코드는 원이 좌우로 움직이는 걸 볼 수 있습니다. 

# 원리
GetMessage 함수는 WM_QUIT 메시지를 만날 때까지 반환하지 않지만 PeekMessage 함수는 즉시 반환되기 때문입니다.



프로젝트 깃헙 
::github{repo="Ushio-Hayase/WinAPI-MessageRoop"}