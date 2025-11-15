---
title: C++ 상속과 캐스팅
published: 2025-11-15
description: 'private, protected, public 상속과 업, 다운 캐스팅'
tags: ["C++"]
category: ''
draft: false 
lang: ''
---

## C++의 상속의 종류

C++에서는 상속을 다음과 같이 한다.

```cpp
class Derived : [virtual] [access-specifier] Base1, 
[virtual] [access-specifier] Base2, ...
```

이 자식 클래스가 부모 클래스를 상속할 때 액세스 지정자에는 3가지 종류가 있다.
public, protected, private 상속이 있는데 단순하게 생각하면 객체지향 관점에서 상속은 모두 is-a,
그러니까 "Base는 Derived이다"라고 생각할 수 있다.

하지만 C++에서는 아니다.

C++에서는 public 상속, 그러니까 `class Derived : public Base`는 "is-a"가 맞지만
나머지 private, protected 상속은 "is-a"가 아닌 "is-implemented-in-terms-of", 그러니까
*Base로 구현된 Derived*라는 것이다.

이 관계는 캐스팅에도 영향을 끼친다.

public 상속은 `Base -> Derived`, `Derived -> Base` 두 방향 모두 어디서든 가능하지만 private, protected 상속은
Derived 클래스의 메서드 내에서를 제외하면 외부에서는 양 방향 캐스팅 모두 허용하지 않는다.

따라서 외부에서 캐스팅을 할 필요가 있다면 public으로 상속을 해야한다.
