---
title: C++ 내 가상 상속과 다중 상속에서의 메서드 호출
published: 2026-01-05
description: '다중 상속과 가상 상속에서의 vtable이 어떻게 호출되는지 알아보기'
tags: ["C++"]
category: 'Knowledge'
draft: false
lang: 'ko'
---

## 다중 상속

### 다중 상속에서 메모리 구조

다중 상속이란 하나의 클래스가 두 개 이상의 부모 클래스를 상속받는 구조입니다.
이 때 객체 내부의 부모 클래스들은 선언된 순서대로 배치됩니다.

각 부모 클래스가 가상 함수를 가지고 있다면 자식 클래스 안에는 부모의 수만큼 vptr(가상 함수 테이블 포인터)가 생성됩니다.

```cpp
class Base1 { void f1() {} virtual void funcA() {} int b1; };
class Base2 { virtual void funcB() {} int b2; };
// Base1과 Base2를 동시에 상속받음
class Derived : public Base1, public Base2 { int d; void funcA() override; void funcB() override; void fNew(); };
```

예시로 위와 같은 상속에서 `Derived` 객체는 아래와 같은 메모리 구조를 갖습니다.

```plaintext
[ Derived 객체의 시작 주소 (Ex: 0x1000) ]
+----------------------------+ <--- (A) Base1* 로 가리킬 때의 주소
| [vptr_Primary] (8 bytes)     | -> Primary vtable을 가리킴
| Base1의 멤버변수 b1 (4 bytes)|
+----------------------------+ <--- (B) Base2* 로 가리킬 때의 주소 (Ex: 0x100C)
| [vptr_Base2] (8 bytes)     | -> Base2의 vtable을 가리킴
| Base2의 멤버변수 b2 (4 bytes)|
+----------------------------+
| Derived의 멤버변수 d (4 bytes)|
+----------------------------+

[ Derived의 Primary VTable ] (0x2000 번지라고 가정)
+-----------------------+
| RTTI (Derived 정보)    |
+-----------------------+
| [0]: Base1::f1()      | <--- Base1에서 상속 (변경 없음)
| [1]: Derived::funcA()    | <--- Base1 슬롯을 Derived가 오버라이드 (주소 교체)
+-----------------------+
| [2]: Derived::funcB()    | <--- Base2에서 온 함수를 최적화를 위해 추가 (Direct Call용)
| [3]: Derived::fNew()  | <--- Derived가 새로 만든 가상 함수
+-----------------------+
```

위 그림에서 중요하게 볼 지점은 Base1으로 가르킬 때의 주소와 Base2로 가르킬 때의 주소가 다르다는 것입니다.

`Derived* d = new Derived();`라고 할 때는 `d`는 당연히 Derived 객체의 시작 주소인 `0x1000`을 가르킵니다.
또 `Base1* b1 = d;`와 같이 가장 먼저 상속된 클래스의 자료형으로 가르킬 때도 가장 앞에 있으니 똑같이 '0x1000`을 가르킵니다.
하지만 `Base2* b2 = d;`와 같이 두 번쨰 이상으로 상속된 클래스의 자료형으로 가르킬 때는 컴파일러가 `d`가 가르키는 주소에 `sizeof(Base1)` 만큼 오프셋을 더해 `0x100C`를 가르키게 합니다.

### Thunk

위와 같이 다중 상속을 했을 경우 중 가상 함수를 오버라이딩한 경우 아래와 같은 상황이 나타날 수 있습니다.

`Base2* b2 = new Derived();`와 같이 `Derived`가 `Base2`의 `funcB()`를 오버라이드했다고 가정했을때 `funcB()`를 `Base2`의 포인터를 통해 호출하려는 상황입니다.
하지만 이 때 `Base2*`가 가르키는 주소는 `Derived*`로 가르킬 때 주소보다 `sizeof(Base1)`만큼 커진 `0x100C`를 가르키는 상태입니다.
하지만 실제 구현인 `Derived::funcB()`는 객체의 실제 시작점인 `0x1000`을 기대합니다.
이를 해결하기 위해 `Base2`용 `vtable`에는 `funcB()`가 들어갈 슬롯에 함수 주소 대신 `Thunk`라는 코드 조각을 가르킵니다.
이 `Thunk`는 실제 함수 주소가 아닌 `this`의 주솟값에서 `sizeof(Base1)`만큼을 뺀 뒤에 실제 함수 주소로 점프하는 명령을 가집니다.

```plaintext
[상황: Base2 포인터(p2)가 Derived 객체의 Base2 영역을 가리키고 있음]

   p2 (주소 0x100C)
      |
      V
+-> [ vptr_Base2 ] -> [ Base2용 vtable ]
|                      +---------------------+
|                      | ...                 |
|                      | funcB의 슬롯         | --> [ Thunk (조정자 코드) ] 로 점프!
(현재 this)            +---------------------+             |
                                                          | (1. this 포인터에서 오프셋 0x0C를 뺌)
                                                          | (2. 이제 this는 0x1000을 가리킴)
                                                          V
                                                    [ 진짜 Derived::funcB() 함수 본체 ]
```

## 가상 상속

C++ 클래스의 다중 상속에서는 일명 `죽음의 다이아몬드(Dreadful Diamond)`와 같은 상황이 일어날 수 있다.

```plaintext
+------------+
      |   Base     |  <-- 최상위 클래스 (예: Animal)
      | (member x) |
      +------------+
       /          \
      /            \
+------------+      +------------+
|  DerivedA  |      |  DerivedB  |  <-- 중간 클래스 (예: Tiger, Lion)
|            |      |            |
+------------+      +------------+
      \            /
       \          /
      +------------+
      |   Final    |  <-- 최하위 다중 상속 클래스 (예: Liger)
      |            |
      +------------+
```

이 상황에서는 `Final` 클래스의 인스턴스가 `Base` 클래스에 정의된 메서드나 변수에 접근할 때 `DerivedA`를 통할지 `DerivedB`를 통할지 판단할 수 없습니다.
따라서 이 상황을 해결하기위해 C++은 가상 상속을 지원합니다.

가상 상속은 중간 단계 클래스(Derived...)들이 상속받을때 `virtual` 키워드를 사용하면, `Final` 클래스에서 `Base` 클래스의 인스턴스를 1개만 유지합니다.

```cpp
class Base { int g; };
class Derived1 : virtual public Base { int p1; }; // 가상 상속
class Derived2 : virtual public Base { int p2; }; // 가상 상속
class Final : public Derived1, public Derived2 { int c; };
```

이 가상 상속을 사용할 시 Final 클래스의 메모리 구조는 공유되는 부모는 가장 뒤로 빼는 식으로 다음과 같이 변합니다.

```plaintext
[ Final 객체의 시작 주소 ]
+-----------------------------------+
| [vptr_P1] (가상 함수용)            |
| [vbptr_P1] (가상 기본 클래스 포인터)| ---> [ Derived1의 vbtable ]
| Parent1의 멤버 p1                  |      (Grand까지 오프셋: +24 저장됨)
+-----------------------------------+
| [vptr_P2]                         |
| [vbptr_P2]                        | ---> [ Derived2의 vbtable ]
| Derived2의 멤버 p2                  |      (Grand까지 오프셋: +12 저장됨)
+-----------------------------------+
| Final의 멤버 c                     |
+===================================+ <--- 여기가 분기점!
| ** 공유된 Base 객체 영역 ** | (객체의 가장 마지막에 단 하나만 존재)
| Base의 멤버 g                     |
+-----------------------------------+
```

이 때 Base 멤버를 접근하면 다음과 같은 과정을 거칩니다.

1. 현재 영역의 vbptr을 읽는다.
2. vbptr이 가리키는 테이블(vbtable)로 간다.
3. 테이블에서 Base까지의 오프셋(거리) 값을 읽는다.
4. 현재 주소에 그 오프셋을 더해서 실제 Base의 위치를 찾아간다.

## 객체 슬라이싱

위와 같은 상황들은 모두 자료형이 포인터형인 경우였습니다.
하지만 값 형태로 객체를 받으면 데이터가 달라집니다.

`Base2 b2 = *new Derived(); // 혹은 Derived d; Base2 b2 = d;`

위와 같이 포인터 형태가 아닌 값 형태로 받을 경우 위 Derived 객체의 멤버 중 Base2가 아닌 요소는 모두 잘려나가고 Base2 부분의 데이터만 복사해서 스택에 할당합니다.
