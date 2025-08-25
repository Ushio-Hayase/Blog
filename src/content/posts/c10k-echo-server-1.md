---
title: C10K 에코 서버 제작 - 1
published: 2025-08-25
description: '동시에 1만개의 연결이 가능한 에코 서버 제작'
tags: ["C++", "Network"]
category: 'Project'
draft: false
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

epoll에 대해 자세히 알려면 만들어진 원인부터 알아보면 더 쉽게 알 수 있습니다.
epoll이전에는 select와 poll이라는 것이 있어 이것들로 여러개의 I/O 작업을 관리했습니다.

select는 감시할 FD의 개수가 FD_SETSIZE(보통 1024)로 제한되고 fd_set이라는 비트맵 형태의 자료구조를 사용하는데 매 호출시마다 이 구조체를 유저 공간에서 커널 공간으로 복사해야했습니다.
또한 커널은 이벤트 발생 여부와 무관하게 모든 FD를 순차적으로 스캔하여 확인해 감시 대상 수-N에 비례하는 O(N)의 시간 복잡도를 가졌습니다.

poll도 비슷하게 감시할 FD 숫자만 해결했지 나머지 문제는 비슷했습니다.

epoll은 이 문제들을 해결하기 위해 상태 관리를 커널에 위임하고 유저는 결과만 통보받도록 바꿨습니다.

epoll을 사용하기 위해선 먼저 `epoll_create1()`을 이용해 epoll 인스턴스를 생성합니다.
이 epoll 인스턴스는 내부적으로 레드-블랙 트리로 구성된 Interest List(관심 목록)과 링크드 리스트로 구성된 Ready List(준비 목록)으로 관리합니다.

이 epoll 인스턴스의 관심 목록을 관리하기 위해서 `epoll_ctl()`을 이용해 관심 목록에서 FD를 추가, 수정, 삭제합니다.

네트워크 디바이스 드라이버가 패킷을 수신하는 등 특정 FD에 이벤트가 발생하면, 커널은 해당 FD가 어느 epoll 인스턴스의 Interest List에 등록되어 있는지 확인합니다.
등록된 FD라면, 커널은 해당 FD를 epoll 인스턴스의 Ready List에 추가하는 콜백 함수를 실행합니다.
이 모든 과정은 커널 내부에서 비동기적으로 일어나며, 유저 프로그램은 관여하지 않습니다.

유저 프로그램은 `epoll_wait()`를 호출하여 Ready List가 비어있지 않을 때까지 대기한 뒤 Ready List에 있는 FD 목록만 유저공간으로 복사하여 반환합니다.
함수가 반환된 뒤, 이벤트가 발생한 소켓의 정보를 담고 있는 `events` 배열을 확인하여 어떤 FD에서 어떤 종류의 이벤트가 발생했는지 알 수 있습니다.

## epoll을 이용한 멀티스레드 동기 에코 서버/클라이언트

이 서버와 클라이언트를 제작하면서 epoll이 정확히 어떤 역할을 맡는지 헷갈려 여러번 수정하며 시도했었는데 그 결과, 이벤트가 발생한 FD가 들어있는 `events` 객체를 반환한다는 걸 어떤건지 이해했습니다.
그래서 그걸 이용한 멀티 스레드 서버를 만들때 원자적 연산을 어떻게 이용해야하는지 이해하고 프로그램을 제작하였습니다.

### 코드 - 서버

```cpp
#include <netinet/in.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <atomic>
#include <cstring>
#include <iostream>
#include <thread>
#include <vector>

constexpr int EVENT_SIZE = 1024;
constexpr int BUF_SIZE = 1024;
constexpr int THREAD_COUNT = 17;

sockaddr_in server_addr;
socklen_t sock_len = sizeof(sockaddr);
std::atomic<int> i{0};
int ev_cnt;
epoll_event ev[EVENT_SIZE];

void worker(int epoll_fd, int listen_sock)
{
    while (true)
    {
        for (; i < ev_cnt; ++i)
        {
            if (ev[i].data.fd == listen_sock && ev[i].events & EPOLLIN)
            {
                sockaddr_in client_addr;

                int acpt_sock =
                    accept(listen_sock, (sockaddr*)&client_addr, &sock_len);

                epoll_event cur_ev;
                cur_ev.data.fd = acpt_sock;
                cur_ev.events = EPOLLIN;

                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, acpt_sock, &cur_ev);
            }
            else
            {
                int acpt_sock = ev[i].data.fd;
                char buf[BUF_SIZE];
                int recv_bytes = recv(acpt_sock, buf, BUF_SIZE, 0);
                if (recv_bytes == -1)
                    std::cerr << "data recving error" << std::endl;

                int send_bytes = send(acpt_sock, buf, BUF_SIZE, 0);
                if (send_bytes == -1)
                    std::cerr << "data sending error" << std::endl;

                close(acpt_sock);
                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, acpt_sock, nullptr);
            }
        }
    }
}

int main()
{
    std::vector<std::thread> thread_pool;

    server_addr.sin_port = htons(8888);
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = 0;
    memset(&(server_addr.sin_zero), 0, 8);

    int epoll_fd = epoll_create1(0);

    int listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_sock == -1) std::cerr << "socket craeting error" << std::endl;
    std::cout << "socket creating" << std::endl;

    int bind_res = bind(listen_sock, (sockaddr*)&server_addr, sock_len);
    if (bind_res == -1) std::cerr << "binding error" << std::endl;
    std::cout << "socket binding" << std::endl;

    int listen_res = listen(listen_sock, EVENT_SIZE);
    if (listen_res == -1) std::cerr << "listening error" << std::endl;
    std::cout << "socket listening start" << std::endl;

    epoll_event listen_ev;
    listen_ev.data.fd = listen_sock;
    listen_ev.events = EPOLLIN;

    int ep_res = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_sock, &listen_ev);
    if (ep_res == -1) std::cerr << "epoll registing error" << std::endl;

    for (int i = 0; i < THREAD_COUNT; ++i)
        thread_pool.emplace_back(worker, epoll_fd, listen_sock);

    while (true)
    {
        int ev_cnt = epoll_wait(epoll_fd, ev, EVENT_SIZE, -1);
        if (i >= ev_cnt) i = 0;
    }
}
```

### 코드 - 클라이언트

```cpp
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <atomic>
#include <cstring>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

constexpr int BUF_SIZE = 1024;
constexpr int MAX_EVENTS = 128;
constexpr int THREAD_COUNT = 17;

std::atomic<int> connection_count{0};
epoll_event events[MAX_EVENTS];
sockaddr_in server_addr;

void worker()
{
    std::string str = "Hello,World!";
    char buf[BUF_SIZE];
    std::fill(buf, buf + BUF_SIZE, '\0');

    for (int i = 0; i < str.size(); ++i) buf[i] = str[i];

    while (true)
    {
        int sock = socket(AF_INET, SOCK_STREAM, 0);
        if (sock == -1) std::cerr << "socket creating error" << std::endl;

        int con_res = connect(sock, (sockaddr *)&server_addr, sizeof(sockaddr));
        if (con_res == -1) std::cerr << "connecting error" << std::endl;

        ++connection_count;

        int res = send(sock, buf, BUF_SIZE, 0);
        if (res == -1) std::cerr << "sending error" << std::endl;

        char recv_buf[BUF_SIZE];

        int recv_res = recv(sock, recv_buf, BUF_SIZE, 0);
        if (recv_res == -1) std::cerr << "recving error" << std::endl;

        close(sock);

        --connection_count;
    }
}

int main()
{
    std::vector<std::thread> thread_pool;

    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8888);

    if (inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr) <= 0)
    {
        std::cerr << "\nInvalid address/ Address not supported \n";
    }

    for (int i = 0; i < THREAD_COUNT; ++i) thread_pool.emplace_back(worker);

    clock_t start = clock();

    while (true)
    {
        clock_t end = clock();
        if ((end - start) / (double)CLOCKS_PER_SEC > 1)
        {
            std::cerr << connection_count << std::endl;
            start = end;
        }
    }
}
```

## 참조

[linux howtos socket 문서](https://www.linuxhowtos.org/C_C++/socket.htm)
