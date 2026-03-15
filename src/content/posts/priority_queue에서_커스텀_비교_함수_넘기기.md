---
title: priority_queue에서 커스텀 비교 함수 넘기기
published: 2025-07-31
description: 'priority_queue에서 커스텀 비교 람다/함수 넘기기'
tags: ["C++"]
category: 'Knowledge'
draft: false 
lang: 'ko'
---

## priority_queue의 선언

priority_queue의 선언은 다음과 같습니다.

```cpp
template< class T, 
class Container = std::vector<T>, 
class Compare = std::less<typename Container::value_type> > 
class priority_queue;
```

## 이유

priority_queue의 세번째 템플릿 매개변수는 타입만을 받습니다.

즉 실제 비교 연산을 할 때 람다함수/함수를 사용한다면 호출할 객체는 따로 넘겨줘야합니다.

왜냐하면 구조체나 클래스는 기본 생성자가 있어 아래 코드의 객체를 초기화할때 기본적으로 초기화가 가능하지만 람다함수/함수는 그게 안되기 때문입니다.

```cpp
template<class T, class Container, class Compare>
class priority_queue {
private:
    Compare comp;  // 비교 객체를 멤버로 저장
    
public:
    priority_queue(const Compare& c) : comp(c) {}  // 생성자에서 초기화
    
    void push(const T& value) {
        // comp를 실제로 호출
        // comp(a, b) 형태로 사용
    }
};
```

Compare에 넘길때 함수에 decltype을 사용하려면 호출이 불가능한 함수 타입이 아닌 함수 포인터 타입을 넘겨야합니다.
`decltype(cmp)` - X , `decltype(&cmp)` - O
