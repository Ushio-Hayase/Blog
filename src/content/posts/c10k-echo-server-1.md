---
title: C10K 에코 서버 제작 - 1
published: 2025-08-08
description: '동시에 1만개의 연결이 가능한 에코 서버 제작'
tags: ["C++", "Network"]
category: 'Project'
draft: true
lang: 'ko'
---

## 리눅스 소켓 프로그래밍 학습

리눅스에서 소켓을 이용해 통신을 할 때는 주로 아래 7개의 함수를 사용합니다.

```cpp
/** 
 * linux clang 18.1.3 x86_64-pc-linux-gnu 컴파일러의 헤더에서 발췌
 * */
1. int socket (int __domain, int __type, int __protocol)
2. int bind (int __fd,__CONST_SOCKADDR_ARG __addr, socklen_t__len)
3. int connect (int __fd,__CONST_SOCKADDR_ARG __addr, socklen_t__len)
4. ssize_t send (int __fd, const void *__buf, size_t __n, int__flags)
5. ssize_t recv (int __fd, void*__buf, size_t __n, int__flags)
6. int listen (int __fd, int__n)
7. int accept (int __fd,__SOCKADDR_ARG __addr, socklen_t *__restrict __addr_len)
```

이 함수들은 `<sys/socket.h>` 헤더에 들어있습니다.

먼저 `socket()` 함수는 순서대로 소켓의 주소 종류, 연결 타입, 소켓의 프로토콜을 인자로 받습니다.

첫 번째 인자가 받을 수 있는 인자로는 다음과 같은 매크로 상수가 있습니다.

| 이름              | 설명                                                                                                                                         |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| AF_UNIX, AF_LOCAL | 운영체제 내 통신                                                                                                                             |
| AF_INET           | IPv4 인터넷 프로토콜                                                                                                                         |
| AF_INET6          | IPv6 인터넷 프로토콜                                                                                                                         |
| AF_IPX            | IPX  Novell 프로토콜                                                                                                                         |
| AF_NETLINK        | 커널과 유저 공간 사이의 정보전송                                                                                                             |
| AF_X25            | [ITU-T X.25](https://en.wikipedia.org/wiki/X.25) / 프로토콜 스위칭 데이터 통신                                                               |
| AF_AX25           | [AX.25 protocol](https://en.wikipedia.org/wiki/AX.25) / X.25의 레이어 2에서 파생된 데이터 링크 계층 프로토콜                                 |
| AF_ATMPVC         | ATM([Asynchronous Transfer Mode](https://en.wikipedia.org/wiki/Asynchronous_Transfer_Mode)) 네트워크에서 사용되는 가상 회선(Virtual Circuit) |
| AF_APPLETALK      | 애플에서 사용하던 네트워킹 프로토콜 [wikipedia](https://en.wikipedia.org/wiki/AppleTalk)                                                     |
| AF_PACKET         | 저수준의 패킷 인터페이스                                                                                                                     |
| AF_ALG            | 커널의 crypto API 사용                                                                                                                       |

두 번째 인자가 받을 수 있는 인자로는 다음 두 종류의 매크로 상수가 있고 bitwise OR 연산자로 두 종류를 동시에 사용할 수 있습니다.

| 이름           | 설명                                                                  |
| -------------- | --------------------------------------------------------------------- |
| SOCK_STREAM    | 데이터의 순서가 보장되고 데이터에 신뢰성있는 바이트 기반의 연결(TCP)  |
| SOCK_DGRAM     | 데이터 그램을 지원하고 데이터에 신뢰성이 없는 비연결 지향의 통신(UDP) |
| SOCK_SEQPACKET | 순서가 보장되고 데이터에 신뢰성있는 데이터 그램 기반의 연결           |
| SOCKET_RAW     | 네트워크 프로토콜을 직접 제어할 수 있게하는 옵션                      |
| SOCKET_RDM     | 순서가 보자오디지않는 신뢰할 수 있는 데이터 그램 기반의 통신          |
| SOCKET_PACKET  | 저수준의 네트워크 인터페이스에 접근                                   |

| 이름          | 설명                                                          |
| ------------- | ------------------------------------------------------------- |
| SOCK_NONBLOCK | connect(), accept(), send(), recv()를 논블로킹으로 설정       |
| SOCK_CLOEXEC  | exec() 관련 시스템 콜이 새 프로그램을 실행할 때 닫히도록 설정 |

세 번째 인자는 특정한 프로토콜 숫자를 받습니다. 이 프로토콜에 사용될  숫자는 첫 번째 인자에 의해 특정됩니다. 단일 프로토콜만 존재할 경우 0으로 지정할 수 있습니다.

두 번째 `bind()` 함수는 순서대로 소켓 fd, 소켓 주소 구조체, 구조체의 크기를 입력받습니다.
소켓 주소 구조체의 코드는 다음과 같습니다.

```cpp
struct sockaddr {
    sa_family_t sa_family;
    char        sa_data[14];
}
```

이 구조체는 다양한 주소 컨테이너를 제공할 수 있도록 설계되었습니다.
IPv4와 IPv6에서는 sockaddr_in 구조체와 sockaddr_in6 구조체에 내용을 채워넣고 주소를 sockaddr로 캐스팅하는 식으로 전달합니다.

세 번째 `connect()` 또한 `bind()` 함수와 동일한 인수 타입을 받습니다.

네 번째와 다섯 번째 `send()`와 `recv()` 함수(`read()`와 `write()` 함수 또한(전자의 함수와는 flags 유무 차이))는 buffer에 있는 데이터를 n바이트의 길이만큼 보내고 수신합니다.
flags에는 MSG_OOB(out-of-bound 데이터를 전송), MSG_WAITTALL(요청한 크기가 모두 차야 함수를 반환), MSG_DONTWAIT(논블로킹으로 작동)을 넣어줄 수 있습니다.

여섯 번째 `listen()` 함수는 `SOCK_STREAM or SOCK_SEQPACKET` 타입의 소켓 파일 디스크럽터를 첫 번째 인자로 받고 sockfd에 연결 대기할 큐의 길이를 두번째 인자로 받습니다.

마지막 `accept()` 함수 또한 `bind()`, `connect()`와 같은 타입의 인자를 받습니다. 받은 인자 중 sockaddr에는 연결한 상대의 정보가 채워지고 반환값으론 연결된 소켓의 파일 디스크럽터가 반환됩니다.

## epoll

epoll과 

## 참조

[linux howtos socket 문서](https://www.linuxhowtos.org/C_C++/socket.htm)
