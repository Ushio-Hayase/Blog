---
title: C10K 에코 서버 제작
published: 2025-08-26
description: '동시에 1만개의 연결이 가능한 에코 서버 제작'
tags: ["C++", "Network"]
category: 'Project'
draft: false
lang: 'ko'
---

## 동기에서 비동기로

저번에 만들었던 에코 서버/클라이언트는 클라이언트에서 송수신하는 부분을 epoll을 사용하지않고 동기적으로 제작하여 총 연결 개수가 클라이언트의 스레드 개수를 넘지 못하게 제작되었습니다.
이제 그 부분을 수정하여 초당 1만개의 연결을 통신할 수 있도록 수정하였습니다.

## 과정

먼저 가장 잘 보이는 클라이언트측의 send와 recv 사이의 딜레이를 줄이기 위해서 클라이언트도 epoll을 도입하고 송신부분과 수신부분을 분리하였습니다.

### 서버 코드

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
            else if (ev[i].events & EPOLLIN)
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
        ev_cnt = epoll_wait(epoll_fd, ev, EVENT_SIZE, -1);
        if (i >= ev_cnt) i = 0;
    }
}
```

### 클라이언트 코드

```cpp
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <atomic>
#include <chrono>
#include <cstring>
#include <iostream>
#include <queue>
#include <thread>
#include <vector>

constexpr int BUF_SIZE = 1024;
constexpr int MAX_EVENTS = 128;
constexpr int THREAD_COUNT = 17;
constexpr int MAX_CONNECT = 12000;

std::atomic<int> connection_count{0};
std::atomic<int> event_counter{0};
int ev_cnt, epoll_fd;
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
        for (; event_counter < ev_cnt; ++event_counter)
        {
            if (events[event_counter].events & EPOLLIN)
            {
                char recv_buf[BUF_SIZE];

                int sock = events[event_counter].data.fd;

                int recv_res = recv(sock, recv_buf, BUF_SIZE, 0);
                if (recv_res == -1) std::cerr << "recving error" << std::endl;

                close(sock);

                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, sock, nullptr);

                --connection_count;
            }
        }

        if (connection_count < MAX_CONNECT)
        {
            int sock = socket(AF_INET, SOCK_STREAM, 0);
            if (sock == -1)
            {
                std::cerr << "socket creating error" << std::endl;
                continue;
            }

            ++connection_count;

            int con_res =
                connect(sock, (sockaddr*)&server_addr, sizeof(sockaddr));
            if (con_res == -1)
            {
                std::cerr << "connecting error" << std::endl;
                close(sock);
                continue;
            }

            int res = send(sock, buf, BUF_SIZE, 0);
            if (res == -1)
            {
                std::cerr << "sending error" << std::endl;
                close(sock);
                continue;
            }

            epoll_event ev;
            ev.data.fd = sock;
            ev.events = EPOLLIN;

            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock, &ev);
        }
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

    epoll_fd = epoll_create1(0);

    for (int i = 0; i < THREAD_COUNT; ++i) thread_pool.emplace_back(worker);

    auto start = std::chrono::system_clock::now();

    while (true)
    {
        auto end = std::chrono::system_clock::now();
        if ((end - start) > std::chrono::seconds(1))
        {
            std::cerr << connection_count << std::endl;
            start = end;
        }
        if (event_counter > MAX_EVENTS)
        {
            ev_cnt = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
            event_counter = 0;
        }
    }
}
```

### 고칠점

일단 위에처럼 코드를 작성하여 WSL상에서 초당 10000개의 연결을 하고 있는 것을 확인했습니다.

하지만 스레드 간의 경쟁 상태를 유발하는 코드가 있고 기타 개선할 점이 있어 코드를 수정하였습니다.

기존의 `events` 배열에 인덱스를 체크하는 변수는 atomic이였지만 실제 접근은 그렇지 않아 문제가 될 수 있습니다.

따라서 이것은 각 스레드가 모두 events 배열를 가지는 것으로 수정해 해결하였습니다.

또 새로운 연결 요청이 들어오면 epoll_wait에서 대기중이던 모든 스레드가 일어나는 문제가 있는데 이것은 listen_sock을 epoll에 등록할 때 EPOLLEXCLUSIVE 플래그를 추가하였습니다.

수정한 코드는 다음과 같습니다.

### 서버 코드

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

void worker(int epoll_fd, int listen_sock)
{
    while (true)
    {
        epoll_event ev[EVENT_SIZE];
        int ev_cnt = epoll_wait(epoll_fd, ev, EVENT_SIZE, -1);

        for (int i = 0; i < ev_cnt; ++i)
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
            else if (ev[i].events & EPOLLIN)
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
    listen_ev.events = EPOLLIN | EPOLLEXCLUSIVE;

    int ep_res = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_sock, &listen_ev);
    if (ep_res == -1) std::cerr << "epoll registing error" << std::endl;

    for (int i = 0; i < THREAD_COUNT; ++i)
        thread_pool.emplace_back(worker, epoll_fd, listen_sock);

    for (int i = 0; i < THREAD_COUNT; ++i) thread_pool[i].join();
}
```

### 클라이언트 코드

```cpp
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <atomic>
#include <chrono>
#include <cstring>
#include <iostream>
#include <queue>
#include <thread>
#include <vector>

constexpr int BUF_SIZE = 1024;
constexpr int MAX_EVENTS = 128;
constexpr int THREAD_COUNT = 17;
constexpr int MAX_CONNECT = 12000;

std::atomic<int> connection_count{0};
std::atomic<int> event_counter{0};
int epoll_fd;

sockaddr_in server_addr;

void worker()
{
    std::string str = "Hello,World!";
    char buf[BUF_SIZE];
    std::fill(buf, buf + BUF_SIZE, '\0');

    for (int i = 0; i < str.size(); ++i) buf[i] = str[i];

    while (true)
    {
        if (connection_count < MAX_CONNECT)
        {
            int sock = socket(AF_INET, SOCK_STREAM, 0);
            if (sock == -1)
            {
                std::cerr << "socket creating error" << std::endl;
                continue;
            }

            ++connection_count;

            int con_res =
                connect(sock, (sockaddr*)&server_addr, sizeof(sockaddr));
            if (con_res == -1)
            {
                std::cerr << "connecting error" << std::endl;
                close(sock);
                continue;
            }

            int res = send(sock, buf, BUF_SIZE, 0);
            if (res == -1)
            {
                std::cerr << "sending error" << std::endl;
                close(sock);
                continue;
            }

            epoll_event ev;
            ev.data.fd = sock;
            ev.events = EPOLLIN;

            epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock, &ev);
        }

        epoll_event events[MAX_EVENTS];
        int ev_cnt = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);

        for (int event_counter = 0; event_counter < ev_cnt; ++event_counter)
        {
            epoll_event events[MAX_EVENTS];
            if (events[event_counter].events & EPOLLIN)
            {
                char recv_buf[BUF_SIZE];

                int sock = events[event_counter].data.fd;

                int recv_res = recv(sock, recv_buf, BUF_SIZE, 0);
                if (recv_res == -1) std::cerr << "recving error" << std::endl;

                close(sock);

                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, sock, nullptr);

                --connection_count;
            }
        }
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

    epoll_fd = epoll_create1(0);

    for (int i = 0; i < THREAD_COUNT; ++i) thread_pool.emplace_back(worker);

    auto start = std::chrono::system_clock::now();

    while (true)
    {
        auto end = std::chrono::system_clock::now();
        if ((end - start) > std::chrono::seconds(1))
        {
            std::cerr << connection_count << std::endl;
            start = end;
        }
    }
}

```

## 고칠점

여기서 connection_count가 중복으로 세지는 문제와 같은 파일 디스크럽터를 여러 스레드가 동시에 접근하는 걸 막기위해 클라이언트는 한번에 소켓을 만든뒤 유지하는 걸로 바꿨습니다.

서버쪽에서도 경쟁 상태가 일어나는 부분이 있었는데 그 부분을 소켓을 epoll에 등록할 때 EPOLLONESHOT과 EPOLLET를 주는 것으로 해결했습니다.

### 서버 코드

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

void worker(int epoll_fd, int listen_sock)
{
    while (true)
    {
        epoll_event ev[EVENT_SIZE];
        int ev_cnt = epoll_wait(epoll_fd, ev, EVENT_SIZE, -1);

        for (int i = 0; i < ev_cnt; ++i)
        {
            if (ev[i].data.fd == listen_sock)
            {
                sockaddr_in client_addr;

                int acpt_sock =
                    accept(listen_sock, (sockaddr*)&client_addr, &sock_len);

                epoll_event cur_ev;
                cur_ev.data.fd = acpt_sock;
                cur_ev.events = EPOLLIN | EPOLLONESHOT | EPOLLET;

                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, acpt_sock, &cur_ev);
            }
            else if (ev[i].events & EPOLLIN)
            {
                int acpt_sock = ev[i].data.fd;
                char buf[BUF_SIZE];
                int recv_bytes = recv(acpt_sock, buf, BUF_SIZE, 0);
                if (recv_bytes > 0)
                {
                    int send_bytes = send(acpt_sock, buf, BUF_SIZE, 0);
                    if (send_bytes == -1)
                        std::cerr << "data sending error: " << strerror(errno)
                                  << std::endl;
                }
                else
                {
                    if (recv_bytes == -1)
                        std::cerr << "data recving error: " << strerror(errno)
                                  << std::endl;

                    close(acpt_sock);
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, acpt_sock, nullptr);
                }
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

    for (int i = 0; i < THREAD_COUNT; ++i) thread_pool[i].join();
}
```

### 클라이언트 코드

```cpp
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <atomic>
#include <chrono>
#include <cstring>
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

constexpr int BUF_SIZE = 1024;
constexpr int MAX_EVENTS = 1024;
constexpr int THREAD_COUNT = 16;
constexpr int MAX_CONNECT = 12000;

std::atomic<int> connection_count{0};
int epoll_fd;

sockaddr_in server_addr;

void worker()
{
    int epoll_fd = epoll_create1(0);
    if (epoll_fd == -1)
    {
        std::cerr << "worker epoll_create1 failed: " << strerror(errno)
                  << std::endl;
        return;
    }

    std::string str = "Hello,World!";
    char send_buf[BUF_SIZE];
    std::fill(send_buf, send_buf + BUF_SIZE, 0);
    memcpy(send_buf, str.c_str(), str.length());

    for (int i = 0; i < MAX_CONNECT / THREAD_COUNT; ++i)
    {
        int sock = socket(AF_INET, SOCK_STREAM, 0);
        if (sock == -1)
        {
            std::cerr << "socket creating error: " << strerror(errno)
                      << std::endl;
            continue;
        }
        if (connect(sock, (sockaddr*)&server_addr, sizeof(sockaddr)) == -1)
        {
            std::cerr << "connecting error: " << strerror(errno) << std::endl;
            close(sock);
            continue;
        }
        if (send(sock, send_buf, BUF_SIZE, 0) == -1)
        {
            std::cerr << "sending error: " << strerror(errno) << std::endl;
            close(sock);
            continue;
        }

        epoll_event ev;
        ev.data.fd = sock;
        ev.events = EPOLLIN;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock, &ev);
        connection_count++;
    }

    epoll_event events[MAX_EVENTS];
    while (true)
    {
        int ev_cnt = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < ev_cnt; ++i)
        {
            if (events[i].events & EPOLLIN)
            {
                int sock = events[i].data.fd;
                char recv_buf[BUF_SIZE];
                int recv_res = recv(sock, recv_buf, BUF_SIZE, 0);

                epoll_ctl(epoll_fd, EPOLL_CTL_DEL, sock, nullptr);
                close(sock);
                --connection_count;

                int new_sock = socket(AF_INET, SOCK_STREAM, 0);
                if (new_sock == -1) continue;
                if (connect(new_sock, (sockaddr*)&server_addr,
                            sizeof(sockaddr)) == -1)
                {
                    close(new_sock);
                    continue;
                }
                if (send(new_sock, send_buf, BUF_SIZE, 0) == -1)
                {
                    close(new_sock);
                    continue;
                }
                epoll_event ev;
                ev.data.fd = new_sock;
                ev.events = EPOLLIN;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, new_sock, &ev);
                ++connection_count;
            }
        }
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

    epoll_fd = epoll_create1(0);

    for (int i = 0; i < THREAD_COUNT; ++i)
    {
        thread_pool.emplace_back(worker);
    }

    auto start = std::chrono::system_clock::now();

    while (true)
    {
        auto end = std::chrono::system_clock::now();
        if ((end - start) > std::chrono::seconds(1))
        {
            std::cerr << connection_count << std::endl;
            start = end;
        }
    }
}
```

## 참고

[github](https://github.com/Ushio-Hayase/c10k-Echo-Server)
