---
title: 프로그램 최적화 방법 소개
published: 2025-03-14
description: 프로그램을 컴파일러 친화적으로 더 빠르게 동작하도록 최적화하는 방법
tags:
  - Computer_Science
category: Knowledge
draft: true
---
# 프로그래머가 최적화를 알아야 하는 이유

현대의 컴파일러는 과거보다 훨씬 더 빠르게 프로그램을 작동하도록 최적화를 시킵니다.
심지어 초보 프로그래머가 최대한 최적화한 코드보다 컴파일러가 *-O1*정도의 수준으로 최적화한 코드가 더 빠르게 실행될 수도 있습니다. 
하지만 컴파일러가 프로그램을 최적화하는데는 한계가 있습니다. 
컴파일러가 과도하게 최적화를 하려다가 언어의 표준이 보장하는 동작을 보장하지 못할 경우가 생길 수 있기에 실제로는 코드 논리 구조와 상관없는 최적화지만 최적화하진 못하는 경우가 생길 수 있기 때문입니다.  

# 최적화 방법
## 루프 비효율성 제거

```cpp
// 데이터 타입을 data_t라고 가정
void func1(data_t* src, data_t* dest)
{
	long i;

	*dest = INITIAL_VALUE; // 초기값 설정

	for (i = 0; i < data_length(src); i++)
	{
		data_t val;
		get_data_from_container(src, i, &val); // 컨테이너로부터 데이터 가져오기
		*dest = *dest + val;
	}
}

void func2(data_t* src, data_t* dest)
{
	long i;
	long length = data_length(src);

	*dest = INITIAL_VALUE; // 초기값 설정

	for (i = 0; i < length; i++)
	{
		data_t val;
		get_data_from_container(src, i, &val); 
		*dest = *dest + val;
	}
}
```

위 코드의 두 함수의 차이는 반복문의 검사를 할 때 쓸 변수를 반복문 내에서 호출하느냐, 아니면 그 전에 초기화하느냐의 차이만 존재합니다.
저와 같은 초보 프로그래머는 두 함수의 속도가 얼마나 차이나느냐고 물을 수 있습니다.
하지만 만약 `data_length`함수의 시간 복잡도가 상수가 아니라면 이야기가 달라집니다. 
대표적으로 문자열과 같은 것의 길이 검사는 보통 $O(n)$의 시간 복잡도가 걸립니다.
즉, 그렇다면 위 `func1`과 `func2`의 시간복잡도는 $O(n^2)$과 $O(n)$으로 시간복잡도가 매우 크게 차이나게 됩니다. 
이런 경우, 컴파일러가 최적화를 해야하느냐 할 수도 있지만 GCC 같은 경우는 반복문 안에서 src가 변할 수도 있기에 최적화를 수행하지 않습니다.
그렇기에, 이런 점을 프로그래머가 잘 캐치하여 최적화하여야 합니다.

## 프로시저 호출 줄이기

```cpp
void get_data_from_container(data_t* src, int i, data_t* dest)
{
	if (i < 0 || i > src->len()) // data_t가 len이란 메소드를 가지고 있다 가정
		throw std::out_of_range();
	*dest = src[i];
}

void func2(data_t* src, data_t* dest)
{
	long i;
	long length = data_length(src);

	*dest = INITIAL_VALUE; // 초기값 설정

	for (i = 0; i < length; i++)
	{
		data_t val;
		get_data_from_container(src, i, &val); 
		*dest = *dest + val;
	}
}

void func3(data_t* src, data_t* dest)
{
	long i;
	long length = data_length(src);
	
	for (i = 0; i < length; i++)
		*dest = *dest + data[i];
}
```

위 코드에서 두 함수는 함수 호출에 따른 오버헤드도 있겠지만 주요 차이점은 인덱스 검사를 하는가 여부입니다.
