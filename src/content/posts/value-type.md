---
title: C++에서의 값 종류
published: 2026-03-26
description: 'C++에서 우측값과 좌측값이 뭔지 알아보고 move, forward까지 알아보기'
image: ''
tags: ["C++"]
category: 'Knowledge'
draft: false 
lang: 'ko'
---

## 기본적인 값 범주

C++의 표현식(expression)은 2개의 독립된 속성인 타입(type)과 값 범주(value category)로 나눌 수 있습니다.
이 표현식은 3가지의 값 범주인 `prvalue`, `xvalue`, `lvalue` 중 하나에 반드시 속합니다.

값 범주는 이 세가지의 기본적인 값 범주와 이것들을 부분집합으로 가지는 값 범주인 `glvalue`, `rvalue`가 있습니다.
이 값 범주들은 다음과 같이 기준을 가지고 정의됩니다.

- identity : 해당 표현식이 메모리 주소를 가져 참조할 수 있는가?
- movability : 해당 표현식의 자원을 다른 곳으로 안전하게 이동할 수 있는가?

|             | have identity | have not identity |     |         |
| ----------- | ------------- | ----------------- | --- | ------- |
| can move    | xvalue        | prvalue           | →   | glvalue |
| cannot move | lvalue        |                   |     |         |
|             | ↓             |                   |     |         |
|             | rvalue        |                   |     |         |

### prvalue

`prvalue`는 내장 연산자의 피연산자로 사용되기위한 순수한 연산의 결과 값이거나 객체를 초기화하는 표현식의 값 범주로 메모리 주소 참조가 불가능하고 (identity = false), 이동이 가능합니다.(movability = true)

`prvalue`의 특징으로는 컴파일시 메모리가 할당되지않고 레지스터에만 머물거나 어셈블리 명령어에 즉치 피연산자(Immediate Operand; 어셈블리에 상수가 직접 포함됌)로 하드코딩되기에 메모리의 주소값을 취할 수 없습니다.
또한 `prvalue`는 불완전한 타입이나 추상 클래스를 가질 수 업고 원시 자료형(primitive type)의 경우 const와 volatile 한정자도 가질 수 없습니다.

`prvalue`는 C++17 이전엔 즉각적으로 임시 객체를 생성하는 것으로 간주되어 불필요한 복사/이동 생성자를 호출할 가능성을 내포했지만 C++17부터는 "객체를 초기화하기위한 방법"으로 변경되었습니다.
따라서 `prvalue`는 그 자체로 객체가 아니고 초기화 문맥에 도달할 때까지 실제 메모리에 객체를 생성하지 않고 `prvalue`를 참조에 바인딩하거나, 클래스의 `prvalue` 멤버에 접근할때 비로소 실체화됩니다.

`prvalue`는 다음과 같은 것들이 있습니다.

- 리터럴(문자열 리터럴 제외) : `42`, `true`, `-17`, `84.f`, `-23.0`, `'a'`
- 후위 증감 연산자의 결과 : `x++`
- 값이 아닌 타입을 반환하는 함수 또는 연산자의 호출의 결과 : `str.substr(1, 2)`, `str1 + str2`
- 산술, 논리, 비트 연산자의 결과 : `a + b`, `a && b,` `a < b`, `a & b`
- this
- 상수 템플릿 표현식의 스칼라 값
- 람다 표현식 그 자체 : `[](int x){ return x * x; }`
- 참조 타입이 아닌 것으로 캐스팅하는 표현식 : `static_cast<double>(x)`
- 주소를 취한 표현식 : `&a`
- 클래스의 열거형 변수(enumerator)나 `static`이 아닌 메서드 참조(데이터 멤버 x) : `a.m`, `p->m`
- 열거형 변수(enumerator)
- 클래스의 멤버 함수 포인터 역참조 : `a.*mp` `a->*mp`
- 콤마 연산자와 삼항 연산자의 우측에 존재하는 표현식들 : `a, b(b가 prvalue)`, `a ? b : c(b, c가 prvalue)`

---

### xvalue

`xvalue`는 곧 수명이 다할 임시객체이거나 프로그래머가 임의로 성질을 바꾼 것으로 메모리 상의 주소를 가지고 있어(identity = true) `lvalue`처럼 취급될 수 있으면서도, 동시에 이동이 가능한(movability = true) 값 범주입니다.

하지만 `xvalue`는 그 정의상 곧 사라질 객체이거나 자원을 뺏겨도 괜찮은 객체이기에 주소 연산자(&)로 주소를 취할 순 없습니다.
`xvalue`의 성질 중 특이한 것은 `xvalue` 표현식이 함수나 연산자에 들어갈때 우측값 참조(T&&) 매개변수을 가진 버전을 최우선적으로 타겟팅하고 만약 없다면 상수 좌측값 참조(const T&)를 우선적으로 타겟팅한다는 것입니다.
`xvalue`는 다른 모든 `rvalue`와 마찬가지로 우측값 참조에 바인딩될 수 있고 다른 `glvalue`와 마찬가지로 다형성을 가질 수 있으며 클래스가 아닌 `xvalue`는 const와 volatile을 가질 수 있습니다.

이러한 `xvalue`에는 다음과 같은 것이 있습니다.

- rvalue 객체의 static이 아닌 객체 데이터 참조 : `a.m(a는 rvalue)`
- 객체 표현식의 멤버 포인터 : `a.*mp`
- 콤마 연산자와 삼항 연산자의 우측에 존재하는 표현식들 : `a, b(b가 xvalue)`, `a ? b : c(b, c가 xvalue)`
- 우측값 참조를 반환하는 함수나 연산자의 표현식 : `std::move(x)`
- rvalue 배열 첨자 연산자 : `a[n](a는 rvalue)`
- 우측값 참조로 캐스팅한 표현식 : `static_cast<char&&>(x)`
- prvalue가 실체화되는 타이밍 (C++17 이상)
- 이동 가능한 표현식(`return 문`, `co_return 문`(C++20 이상), `throw 문`(C++17 이상); C++23 이상)

---

### lvalue

`lvalue`는 메모리 상에서 명확하게 주소를 가지고 있으며(identity = true), 지속적으로 접근할 수 있지만, 자원을 임의로 빼앗을 순 없는(movability = false) 값 범주입니다.
`lvalue`는 다형성을 가지고 주소 연산자(`&`)를 취할 수 있으며 좌측값 참조와 상수 좌측값 참조를 초기화하는데 사용할 수 있고, const가 아닌 lvalue는 대입 연산자의 좌측에 위치할 수 있습니다.

이러한 `lvalue`는 다음과 같은 것이 있습니다.

- 이름있는 변수, 함수, 템플릿 파라미터 오브젝트(C++20 이상) 또는 데이터 멤버, 우측값 참조 변수
- 좌측값 참조를 반환하는 함수나 내장되지 않은 연산자의 표현식 : `std::cout << 1`
- 모든 내장된 대입 연산자 표현식 : `a = b`, `a += b`, `a %= b`
- 전위증감연산자 표현식 : `++x`, `--x`
- 내장된 역참조 표현식 : `*p`
- 내장된 배열 첨자 연산자(C++11 이후는 a[n]이 좌측값일때)
- 우측값, 열거형 변수, static이 아닌 메서드, a가 우측값이고 m이 static이 아닌 데이터 멤버일때를 제외한 객체 멤버 표현식 : `a.m`
- 열거형 변수나 static이 아닌 메서드를 제외한 객체 멤버 포인터 접근 표현식 : `p->m`
- `lvalue` 객체의 멤버 접근후 데이터 변수 역참조 : `a.*mp`
- 내장된 포인터 객체 멤버 접근후 데이터 변수 역참조 : `p->*mp`
- 내장된 콤마 연산자와 삼항 연산자의 우측에 존재하는 표현식들 : `a, b(b가 lvalue)`, `a ? b : c(b, c가 lvalue)`
- 문자열 리터럴 : `"Hello, world!"`
- 좌측값 참조 캐스팅 : `static_cast<int&>(x)`, `static_cast<const int&>(x)`
- 좌측값 참조 템플릿 매개변수 상수
- 우측값 참조를 반환하는 함수를 반환값으로 가지는 함수나 연산자 표현식(C++11 이상)
- 우측값 참조를 반환하는 함수로 캐스팅하는 표현식 : `static_cast<void(&&)(int)>(x)`

---

### const T&

좌측값 참조는 참조를 하는 대상을 변경할 수 없기에 prvalue, xvalue, lvalue 모두를 대상으로 가질 수 있습니다.
따라서 좌측값 참조는 참조를 하는 대상을 자신의 수명과 같게 강제로 연장시킵니다.

---

## glvalue와 rvalue

`glvalue`와 `rvalue`는 위의 기본적인 값 범주들 중 특징이 같은것끼리 모아두고 이름을 붙인 것으로 `glvalue`는 `lvalue`와 `xvalue`를 포함하고 `rvalue`는 `prvalue`와 `xvalue`를 포함합니다.

`glvalue`의 특징적인 점으로는 다형성을 가질 수 있고, 완전하지 않은 타입도 가질 수 있습니다.
또한 `glvalue`는 `lvalue`에서 `rvalue`로 배열에서 포인터로 함수에서 포인터로 암시적으로 전환될 수 있습니다.

`rvalue`는 내장된 주소 연산자를 취하지 못하고 내장된 연산자의 좌측에 오지 못하며 우측값 참조 변수를 초기화하고 자신의 수명을 그 변수의 수명까지 늘릴 수 있습니다.

---

## 참조 축약 규칙(Reference Collapsing Rules)과 완벽한 전달(Perfect Forwarding)

### 참조 축약 규칙

C++ 컴파일러는 문법적으로 참조에 대한 참조를 선언하는 것을 금지합니다.
하지만 `typedef`, `using`, `decltype`을 사용하며 참조에 대한 참조가 발생하는 경우가 있기에 다음과 같은 규칙으로 참조가 축약됩니다.

- `T&` + `&` = `T&`
- `T&` + `&&` = `T&`
- `T&&` + `&` = `T&`
- `T&&` + `&&` = `T&&`

이러한 참조 축약 규칙이 많이 적용되는 공간은 템플릿 매개변수 `T&&`으로 이 인자에 좌측값이 전달되면 컴파일러는 `T`를 `A&`로 추론하여 `A& &&`, 최종적으론 `A&`로 평가됩니다.
거꾸로 우측값이 전달된 경우는 컴파일러는 `T`를 `A`로 추론하여 최종적으론 `A&&`로 평가됩니다.

---

### std::move, std::forward

C++11에서 도입된 std::move와 std::forward는 잘 사용할시 오버헤드를 크게 줄일 수 있는 함수들입니다.
std::move는 인자로 전달된 표현식의 값 범주를 `xvalue`로 변환하여 반환합니다.

```cpp
template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& arg) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(arg);
}
```

이 std::move 함수를 사용하면 좌측값을 `xvalue` 표현식으로 바꿔낼 수 있게되고 그에 따라 컴파일러가 호출할 클래스의 복사 생성자/복사 대입 연산자(const T&) 대신 이동 생성자/이동 대입 연산자(T&&)를 선택하도록 만들 수 있습니다.

이런 std::move 연산의 효과를 잘 보여주는 것이 바로 swap하는 상황입니다.

과거에는 아래같은 함수가 있었으면 복사 생성자/복사 대입 연산자가 호출되지만 현재에는 std::move를 이용하면 이동 생성자/이동 대입 연산자를 호출하여 객체를 복사하지 않고도 swap할 수 있게됩니다.

```cpp
template<typename T>
void swap(T& lhs, T& rhs)
{
    T tmp(lhs); // 복사 생성자
    lhs = rhs; // 복사 대입 연산자
    rhs = lhs; // 복사 대입 연산자
}
```

```cpp
template<typename T>
void swap(T& lhs, T& rhs)
{
    T tmp(std:move(lhs)); // 이동 생성자
    lhs = rhs; // 이동 대입 연산자
    rhs = lhs; // 이동 대입 연산자
}
```

---

### 복사 생략(Copy Elision)

복사 생략(Copy Elision)은 RVO(Return Value Optimization)[^rvo]와 NRVO(Named Return Value Optimization)[^nrvo]를 이용해 함수가 객체를 반환할때 이동/복사 생성자 호출을 생략하는 최적화입니다.

이 때 주의할 것으로 함수가 반환값을 돌려줄 때 아래와 같이 std::move()를 이용하여 이동으로 반환값을 줘 오버헤드를 줄이자는 생각을 할 수도 있는데
`xvalue`로 바뀌면 `lvalue`여야 작동하는직 NRVO를 사용하지 못하기에 오히려 속도가 느려질 수 있습니다.

[^rvo]: RVO - `return Object()`와 같이 `prvalue`가 반환될 경우 임시 객체를 생성하지않고 함수를 호출하여 반환값을 수신하는 객체의 메모리에 객체를 직접 생성하는 최적화 기술, 함수의 반환 타입과 `return` 문의 명시된 객체의 타입이 정확히 일치하고 반환되는 객체가 `prvalue`이면 적용됩니다.

[^nrvo]: NRVO - 위의 RVO를 확장하여 함수 내부의 지역변수를 반환할 때 해당 지역변수를 함수의 스택이 아닌 반환값을 수신하는 객체의 메모리에 직접 객체를 생성하는 최적화 기술, 함수의 반환 타입과 `return` 문의 명시된 객체의 타입이 정확히 일치하고 반환되는 객체가 지역변수이며 함수의 모든 실행 경로에서 동일한 지역 변수가 반환되거나 컴파일러가 반환될 변수의 메모리 주소를 추적할 수 있어야 적용됩니다.

---

## 완벽한 전달(Perfect Forwarding)

템플릿 프로그래밍에서는 외부로부터 인자를 받아 내부 함수로 전달하는 래퍼(wrapper) 함수를 작성할 때 값 범주에 대한 문제가 생깁니다.
아래와 같은 함수가 있을 때 인자에 `prvalue`를 전달했더라도 내부에서 `arg`라는 이름이 부여되어 `lvalue`로 바뀌어 버립니다.

```cpp
template <typename T>
void wrapper(T arg) {
    internal_func(arg);
}
```

이런 상황을 막기위해 호출자가 넘겨준 인자의 특성을 보존하여 내부 함수로 전달하는 기법이 *완벽한 전달*입니다.

```cpp
template <typename T>
constexpr T&& forward(std::remove_reference_t<T>& arg) noexcept {
    return static_cast<T&&>(arg);
}
```

std::forward 함수는 위와 같은 형태로 std::move가 무조건적인 `rvalue` 캐스팅을 하는것에 반해 조건부로 캐스팅을 수행합니다.
호출자가 `lvalue`를 넘겼을 때는 `std::forward<T&>`가 호출이 되고 `static_cast<T& &&>`로 치환되어 최종적으론 `static_cast<T&>`가 됩니다.
호출자가 `rvalue`를 넘기면 `std::forward<T>`가 호출이 되고 최종적으론 `static_cast<T&&>`가 호출되어 이동 관련 함수를 호출할 수 있게 됩니다.

## 참조

cppreference - [link](https://en.cppreference.com/w/cpp/language/value_category)  
모두의 코드 - [link](https://modoocode.com/189)
