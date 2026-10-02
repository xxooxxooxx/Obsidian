---
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^review|复习]]
  - [[#^c|C 语言]]
    - [[#^memory-and-strings|内存与字符串]]
    - [[#^printing|打印]]
    - [[#^input-parsing|输入解析]]
  - [[#^processes|进程]]
  - [[#^memory|内存]]
  - [[#^threading-and-synchronization|线程与同步]]
  - [[#^deadlock|死锁]]
  - [[#^ipc|进程间通信]]
  - [[#^filesystems|文件系统]]
  - [[#^networking|网络]]
  - [[#^security|安全]]
  - [[#^signals|信号]]


# 复习 ^review

下面是一份并不完整的知识点清单。

## C 语言 ^c

### 内存与字符串 ^memory-and-strings

1.  在下面的例子中，哪些变量保证会打印出零值？

    ``` objectivec
    int a;
    static int b;

    void func() {
      static int c;
      int d;
      printf("%d %d %d %d\n",a,b,c,d);
    }
    ```

2.  在下面的例子中，哪些变量保证会打印出零值？

    ``` objectivec
    void func() {
      int* ptr1 = malloc(sizeof(int));
      int* ptr2 = realloc(NULL, sizeof(int));
      int* ptr3 = calloc(1, sizeof(int));
      int* ptr4 = calloc(sizeof(int), 1);

      printf("%d %d %d %d\n",*ptr1,*ptr2,*ptr3,*ptr4);
    }
    ```

3.  解释下面这段复制字符串的尝试中的错误。

    ``` objectivec
    char* copy(char*src) {
      char*result = malloc( strlen(src) );
      strcpy(result, src);
      return result;
    }
    ```

4.  为什么下面这段复制字符串的尝试有时能工作、有时会失败？

    ``` objectivec
    char* copy(char*src) {
      char*result = malloc( strlen(src) +1 );
      strcat(result, src);
      return result;
    }
    ```

5.  解释下面这段试图复制字符串的代码中的两处错误。

    ``` objectivec
    char* copy(char*src) {
      char result[sizeof(src)];
      strcpy(result, src);
      return result;
    }
    ```

6.  以下哪一项是合法的？

    ``` objectivec
    char a[] = "Hello"; strcpy(a, "World");
    char b[] = "Hello"; strcpy(b, "World12345", b);
    char* c = "Hello"; strcpy(c, "World");
    ```

7.  补全函数指针 typedef，声明一个接受 void\* 参数并返回 void\* 的函数指针。把你的类型命名为 ‘pthread_callback’

    ``` objectivec
    typedef ______________________;
    ```

8.  除了函数参数之外，线程的栈上还存放了什么？

9.  仅用 `char* strcat(char*dest, const char*src)`、`strcpy`、`strlen` 和指针算术，实现一个版本

    ``` objectivec
    char* mystrcat(char*dest, const char*src) {

      ? Use strcpy strlen here

      return dest;
    }
    ```

10. 用循环、且不调用任何函数，实现一个 size_t strlen(const char\*) 的版本。

    ``` objectivec
    size_t mystrlen(const char*s) {

    }
    ```

11.  找出下面这份 `strcpy` 实现中的三个 bug。

    ``` objectivec
    char* strcpy(const char* dest, const char* src) {
      while(*src) { *dest++ = *src++; }
      return dest;
    }
    ```

### 打印 ^printing

1.  找出这两处错误！

    ``` objectivec
    fprintf("You scored 100%");
    ```

2.  补全下面的代码，把姓名、一个逗号和分数打印到文件 ‘result.txt’ 中

    ``` objectivec
    char* name = .....;
    int score = ......
    FILE *f = fopen("result.txt",_____);
    if(f) {
      _____
    }
    fclose(f);
    ```

3.  你会怎样把变量 `a`、`mesg`、`val` 和 `ptr` 的值打印成一个字符串？把 a 打印为整数，mesg 打印为 C 字符串，val 打印为 double，ptr 打印为十六进制指针。你可以假定 mesg 指向一个较短的 C 字符串（\<50 个字符）。加分项：你怎样让这段代码更健壮，从而既能应付任意长度的 `mesg`，也能应付 `malloc` 返回 `NULL` 的情况？提示：用 `%f` 打印一个很大的 `val` 可能要输出 300 多个字符。

    ``` objectivec
    char* toString(int a, char*mesg, double val, void* ptr) {
      char* result = malloc( strlen(mesg) + 50);
      _____
      return result;
    }
    ```

### 输入解析 ^input-parsing

1.  为什么你应该检查 sscanf 和 scanf 的返回值？

2.  为什么 `gets` 很危险？

3.  写一个完整的程序，使用 `getline`。确保你的程序没有内存泄漏。

4.  你会在什么时候用 calloc 而不是 malloc？realloc 在什么时候会有用？

5.  程序员在下面这段代码中犯了什么错？有可能修复吗？

    1.  使用堆内存的情形？

    2.  使用全局（静态）内存的情形？

    ``` objectivec
    static int id;

    char* next_ticket() {
      id ++;
      char result[20];
      sprintf(result,"%d",id);
      return result;
    }
    ```

## 进程 ^processes

1.  什么是进程？

2.  fork 时会有哪些属性从原进程传递给子进程？exec 调用成功时呢？

3.  什么是 fork 炸弹？我们要如何避免它？

4.  wait 系统调用用来做什么？

5.  什么是僵尸进程？我们如何避免它们？

6.  什么是孤儿进程？它们会怎样？

7.  我们如何检查一个已经退出的进程的状态？

8.  进程的一种常见模式是什么？

## 内存 ^memory

1.  C 语言中用来分配内存的调用有哪些？

2.  malloc 分配的内存必须按什么对齐？为什么这很重要？

3.  什么是 Knuth 分配方案？

4.  在伙伴分配方案中，你会如何处理一次分配请求？

5.  什么是空闲链表（free list）？

6.  向空闲链表中插入有哪几种不同的做法？

7.  首次适配、最差适配、最佳适配各自有哪些优点和缺点？

8.  下面这个简陋的 malloc 实现，在什么情况下是可以接受的？

    ``` objectivec
    void *malloc(int size) {
      return (void *)sbrk(size);
    }
    ```

## 线程与同步 ^threading-and-synchronization

1.  什么是线程？线程之间共享什么？

2.  如何创建一个线程？

3.  线程的栈位于内存中的什么位置？

4.  什么是互斥锁？它解决了什么问题？

5.  什么是条件变量？它解决了什么问题？

6.  写一个线程安全的链表，支持在头部插入、尾部插入、从头部弹出、从尾部弹出。确保它不会忙等（busy wait）！

7.  Peterson 针对临界区问题的解法是什么？Dekker 的呢？

8.  下面的代码是线程安全的吗？请重新设计它，使其线程安全。提示：如果那块消息内存对每次调用而言是独有的，那么互斥锁就不必要。

    ``` objectivec
    static char message[20];
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

    void *format(int v) {
      pthread_mutex_lock(&mutex);
      sprintf(message, ":%d:" ,v);
      pthread_mutex_unlock(&mutex);
      return message;
    }
    ```

9.  以下哪一种情况可能让进程继续运行下去？

    1.  在最后一个运行中的线程里，从 pthread 的起始函数返回。

    2.  原始线程从 main 返回。

    3.  任何线程引发段错误。

    4.  任何线程调用 `exit`。

    5.  在其他线程仍在运行时，在主线程中调用 `pthread_exit`。

10. 为下面这个程序打印出的 “W” 字符数量写出一个数学表达式。假定 a、b、c、d 都是小的正整数。你的答案可以使用一个返回其最小参数的 ‘min’ 函数。

    ``` objectivec
    unsigned int a=...,b=...,c=...,d=...;

    void* func(void* ptr) {
      char m = * (char*)ptr;
      if(m == 'P') sem_post(s);
      if(m == 'W') sem_wait(s);
      putchar(m);
      return NULL;
    }

    int main(int argv, char** argc) {
      sem_init(s,0, a);
      while(b--) pthread_create(&tid, NULL, func, "W");
      while(c--) pthread_create(&tid, NULL, func, "P");
      while(d--) pthread_create(&tid, NULL, func, "W");
      pthread_exit(NULL);
      /*Process will finish when all threads have exited */
    }
    ```

11. 补全下面的代码。下面这段代码本应交替打印 `A` 和 `B`，它表示两个线程轮流执行。给 `func` 添加条件变量调用，使等待的线程不必持续检查 `turn` 变量。问：pthread_cond_broadcast 是必要的，还是 pthread_cond_signal 就够了？

    ``` objectivec
    pthread_cond_t cv = PTHREAD_COND_INITIALIZER;
    pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

    void* turn;

    void* func(void* mesg) {
      while(1) {
        // Add mutex lock and condition variable calls ...

        while(turn == mesg) {
          /* poll again ... Change me - This busy loop burns CPU time! */
        }

        /* Do stuff on this thread */
        puts( (char*) mesg);
        turn = mesg;

      }
      return 0;
    }

    int main(int argc, char** argv){
      pthread_t tid1;
      pthread_create(&tid1, NULL, func, "A");
      func("B"); // no need to create another thread - use the main thread
      return 0;
    }
    ```

12. 找出给定代码中的临界区。加入互斥锁加锁使代码线程安全。添加条件变量调用，使 `total` 永远不会变成负数或超过 1000；相反，该调用应当阻塞，直到可以安全继续为止。解释为什么 `pthread_cond_broadcast` 是必要的。

    ``` objectivec
    int total;
    void add(int value) {
      if(value < 1) return;
      total += value;
    }
    void sub(int value) {
      if(value < 1) return;
      total -= value;
    }
    ```

13. 某个非线程安全的数据结构有 `size()`、`enq` 和 `deq` 方法。请使用条件变量和互斥锁，补全其线程安全、阻塞式版本。

    ``` objectivec
    void enqueue(void* data) {
      // should block if the size() would become greater than 256
      enq(data);
    }
    void* dequeue() {
      // should block if size() is 0
      return deq();
    }
    ```

14. 你们公司的启动项目要利用最新交通信息做路径规划。你那位薪酬过高的实习生写了一个非线程安全的数据结构，包含两个函数：`shortest`（使用但不修改该图）和 `set_edge`（会修改该图）。

    ``` objectivec
    graph_t* create_graph(char* filename); // called once

    // returns a new heap object that is the shortest path from vertex i to j
    path_t* shortest(graph_t* graph, int i, int j);

    // updates edge from vertex i to j
    void set_edge(graph_t* graph, int i, int j, double time);
    ```

    For performance, multiple threads must be able to call `shortest` at
    the same time, but the graph can only be modified by one thread when
    no other threads are executing inside `shortest` or `set_edge`. Use
    a mutex lock and condition variables to implement a reader-writer
    solution. An incomplete attempt is shown below. Though this attempt
    is thread safe (thus sufficient for demo day!), it does not allow
    multiple threads to calculate `shortest` path at the same time and
    will not have sufficient throughput.

    ``` objectivec
    path_t* shortest_safe(graph_t* graph, int i, int j) {
      pthread_mutex_lock(&m);
      path_t* path = shortest(graph, i, j);
      pthread_mutex_unlock(&m);
      return path;
    }
    void set_edge_safe(graph_t* graph, int i, int j, double dist) {
      pthread_mutex_lock(&m);
      set_edge(graph, i, j, dist);
      pthread_mutex_unlock(&m);
    }
    ```

15. 就读者-写者问题而言，下面这些说法有多少条是正确的？

    - 可以有多个活动的读者

    - 可以有多个活动的写者

    - 当存在一个活动的写者时，活动的读者数量必须为零

    - 当存在一个活动的读者时，活动的写者数量必须为零

    - 写者必须等到当前活动的读者全部完成

## 死锁 ^deadlock

1.  Coffman 条件有哪些？它们分别是什么意思？你能给出每一条件的定义，以及一个用互斥锁破坏该条件的例子吗？

2.  逐一给出破坏每个 Coffman 条件的现实例子。可以考虑这样一个场景：画工、油漆和画笔。

    1.  持有并等待

    2.  循环等待

    3.  不可剥夺

    4.  互斥

3.  判断哲学家就餐代码在何时会导致死锁（或不会）。例如，若你看到下面这段代码片段，它不满足哪个 Coffman 条件？

    ``` objectivec
    // Get both locks or none.
    pthread_mutex_lock( a );
    if( pthread_mutex_trylock( b ) ) { /*failed*/
      pthread_mutex_unlock( a );
      ...
    }
    ```

4.  有多少个进程处于阻塞状态？

    - P1 获取 R1

    - P2 获取 R2

    - P1 获取 R3

    - P2 等待 R3

    - P3 获取 R5

    - P1 获取 R4

    - P3 等待 R1

    - P4 等待 R5

    - P5 等待 R1

5.  对于哲学家就餐问题的以下几种解法，各有哪些优缺点

    1.  仲裁者（Arbitrator）

    2.  Dijkstra

    3.  Stalling’s

    4.  Trylock

## 进程间通信 ^ipc

1.  以下各项是什么？它们的用途是什么？

    1.  翻译后备缓冲器（Translation Lookaside Buffer）

    2.  物理地址

    3.  内存管理单元

    4.  脏位（dirty bit）

2.  你如何确定页内偏移占用了多少位？

3.  上下文切换 20ms 之后，TLB 中已经装入了你那段数值代码所用的全部逻辑地址，而这段代码 100% 的时间都在访问主存。相比单级页表，两级页表带来的额外开销（变慢程度）是多少？

4.  解释为什么在发生上下文切换时（即 CPU 被分配去处理另一个进程时）必须刷新 TLB。

5.  填空，使下面这个程序打印出 123456789。如果 `cat` 没有给定参数，它就只是把输入一直打印到 EOF。加分项：解释为什么下面的 `close` 调用是必要的。

    ``` objectivec
    int main() {
      int i = 0;
      while(++i < 10) {
        pid_t pid = fork();
        if(pid == 0) { /* child */
          char buffer[16];
          sprintf(buffer, ______,i);
          int fds[ ______];
          pipe(fds);
          write(fds[1], ______,______ ); // Write the buffer into the pipe
          close(______);
          dup2(fds[0], ______);
          execlp("cat", "cat",  ______);
          perror("exec"); exit(1);
        }
        waitpid(pid, NULL, 0);
      }
      return 0;
    }
    ```

6.  使用 POSIX 调用 `fork`、`pipe`、`dup2` 和 `close` 实现一个自动评分程序。把子进程的标准输出捕获到管道中。子进程应当用不带任何额外参数（除进程名外）的形式 `exec` 程序 `./test`。在父进程中从管道读取：一旦捕获到的输出中出现 ! 字符，就让父进程退出。退出之前，向子进程发送 SIGKILL。如果输出中包含 !，则退出码为 0。否则，若子进程退出导致管道写端关闭，则以值 1 退出。务必在父进程和子进程中都关闭管道不用的那一端

7.  这道高难度题用管道让一个“AI 玩家”自己对弈，直到游戏结束。程序 `tic tac toe` 接受一行输入——迄今为止的落子序列，打印同样的序列再加上一步，然后退出。一步用两个字符表示。例如 “A1” 和 “C3” 是两个对角的位置。字符串 `B2A1A3` 是一盘 3 步／对弈的棋局。合法应答是 `B2A1A3C1`（C1 这一手封住了 B2 A3 的斜线威胁）。输出行还可以带有后缀 `-I win`、`-You win`、`-invalid` 或 `-draw`。用管道来控制每一个所创建子进程的输入与输出。当输出中出现 `-` 时，打印最终的输出行（完整的棋局序列和结果）并退出。

8.  写一个函数，用 fseek 和 ftell 把一个文件的中间那个字符替换成 ‘X’

    ``` objectivec
    void xout(char* filename) {
      FILE *f = fopen(filename, ____ );

      // Your code here ...
    }
    ```

9.  什么是 MMU？与直接内存访问系统相比，使用它有什么缺点？

10. 什么是管道？

11.  具名管道与匿名管道各有什么优缺点？

## 文件系统 ^filesystems

1.  什么是文件 API？

2.  文件名存储在哪里？

3.  inode 中包含什么？

4.  每个目录中那两个特殊的文件名是什么？

5.  你如何解析下面这个路径 `a/../b/./c/../../c`？

6.  rwx 这三组分别代表什么？

7.  什么是 UID？GID？UID 与 Effective UID 有什么区别？

8.  什么是 umask？

9.  什么是 sticky bit？

10. 什么是虚拟文件系统？

11. 什么是 RAID？

12. 在 `ext2` 文件系统中，为了访问文件 `/dir1/subdirA/notes.txt` 的第一个字节，需要从磁盘读取多少个 inode？假定根目录中的目录名和 inode 号（但不含 inode 本身）已经在内存中。

13. 在 `ext2` 文件系统中，为了访问文件 `/dir1/subdirA/notes.txt` 的第一个字节，最少需要从磁盘读取多少个磁盘块？假定根目录中的目录名和 inode 号、以及所有 inode 都已经在内存中。

14. 在一个地址为 32 位、磁盘块为 4KiB 的 `ext2` 文件系统中，一个 inode 可以存放 10 个直接磁盘块号。要用到一级间接表，所需的最小文件大小是多少？ii) 二级间接表呢？

15.  修正下面的 shell 命令 `chmod`，使文件 `secret.txt` 的权限变为：所有者可读、可写、可执行，所在组只能读，其他人没有任何访问权限。

    ``` bash
    $ chmod 000 secret.txt
    ```

## 网络 ^networking

1.  什么是套接字（socket）？

2.  互联网有哪些不同的层次？

3.  什么是 IP？什么是 IP 地址？

4.  什么是 TCP？什么是 UDP？它们有什么区别？

5.  编写一个向服务器发送 “Hello” 的 TCP 客户端。

6.  编写一个简单的 TCP 回显服务器。它从客户端读取字节直到客户端关闭，然后把字节回显给客户端。

7.  编写一个 UDP 客户端，向 argv\[1\] 指定的主机名洪泛发送数据包。

8.  什么是 HTTP？

9.  什么是 DNS？

10. 为什么网络编程要使用非阻塞 IO？

11. 什么是 RPC？

12. 在 1000 端口上监听和在 2000 端口上监听有什么不同之处？

    - 2000 端口的速率是 1000 端口的一半

    - 2000 端口的速率是 1000 端口的两倍

    - 1000 端口需要 root 权限

    - 没什么区别

13.  描述 IPv4 和 IPv6 之间一个显著的差异。

14.  你会在什么时候、为什么使用 ntohs？

15. 如果一个主机地址是 32 位，我最可能用的是哪种 IP 方案？128 位呢？

16.  哪种常见网络协议是基于数据包的，而且可能无法成功送达数据？

17.  哪种常见协议是基于流的，并且会在数据包丢失时重发数据？

18.  什么是 SYN、SYN-ACK、ACK 三次握手？

19.  以下哪一项**不是** TCP 的特性？

    1.  数据包重排序

    2.  流量控制

    3.  数据包重传

    4.  简单的错误检测

    5.  加密

20. 哪种协议使用了序列号？它们的初始值是什么？为什么？

21.  构建一个 TCP 服务器最少需要哪些网络调用？它们的正确顺序是什么？

22.  构建一个 TCP 客户端最少需要哪些网络调用？它们的正确顺序是什么？

23.  你会在什么时候对一个 TCP 客户端调用 bind？

24.  `socket`、`bind`、`listen` 和 `accept` 的作用是什么？

25.  上述调用中，哪些可能会阻塞、等待新客户端连接？

26. 什么是 DNS？它替你做什么？CS241 中的哪些网络调用会替你使用它？

27.  对 getaddrinfo 而言，你如何指定一个服务器套接字？

28.  getaddrinfo 为什么可能产生网络数据包？

29.  哪个网络调用用于指定允许的 backlog 的大小？

30.  哪个网络调用会返回一个新的文件描述符？

31.  被动套接字在什么时候使用？

32.  什么时候 epoll 比 select 是更好的选择？什么时候 select 比 epoll 更好？

33.  `write(fd, data, 5000)` 是否总是会发送 5000 字节的数据？它什么时候可能失败？

34.  网络地址转换（NAT）是如何工作的？

35.  假设客户端与服务器之间的单向传输时延为 20ms，那么从客户端发出 SYN、到服务器收到最后一个 ACK 为止，TCP 三次握手（SYN、SYN-ACK、ACK）完成总共需要多长时间？

    1.  20ms

    2.  40ms

    3.  100ms

    4.  60ms

36.  HTTP 1.0 与 HTTP 1.1 之间的差异有哪些？如果网络传输时间为 20ms，从服务器向客户端传输 3 个文件需要多少毫秒？HTTP 1.0 与 HTTP 1.1 所需的时间有何不同？

37.  往网络套接字写入时，可能无法把所有字节都发出去，也可能因为信号而被中断。检查 `write` 的返回值，实现 `write_all`，以便用剩余数据反复调用 `write`。如果 `write` 返回 -1，则立即返回 -1，除非 `errno` 等于 `EINTR`——那种情况下应重复上一次 `write` 的尝试。你需要使用指针算术。

    ``` objectivec
    // Returns -1 if write fails (unless EINTR in which case it recalls write
    // Repeated calls write until all of the buffer is written.
    ssize_t write_all(int fd, const char *buf, size_t nbyte) {
      ssize_t nb = write(fd, buf, nbyte);
      return nb;
    }
    ```

38. 实现一个多线程 TCP 服务器，监听 2000 端口。每个线程应从客户端文件描述符读取 128 字节并回显给客户端，然后关闭连接并结束该线程。

39. 实现一个 UDP 服务器，监听 2000 端口。预留 200 字节的缓冲区。监听到达的数据包。合法的数据包不超过 200 字节，并以四个字节 0x65 0x66 0x67 0x68 开头。忽略非法数据包。对合法数据包，把第五个字节的值按无符号数累加到一个累计总和中，并打印到目前为止的总和。如果总和大于 255 就退出。

## 安全 ^security

1.  数据安全的三项措施是什么？

2.  什么是栈破坏（stack smashing）？

3.  什么是缓冲区溢出？

4.  操作系统如何提供安全性？请举出 Networking 和 Filesystems 中的几个例子。

5.  TCP 提供了哪些安全特性？

6.  DNS 是安全的吗？

## 信号 ^signals

1.  说出两个通常由内核产生的信号的名字。

2.  说出一个无法被信号处理程序捕获的信号的名字。

3.  为什么在信号处理程序中调用任何函数（任何并非“信号处理程序安全”的东西）是不安全的？

4.  写一小段代码，使用 SIGACTION 和 SIGNALSET 创建一个 SIGALRM 处理程序。

5.  disposition（处置方式）、mask（掩码）和 pending（待处理）信号集合之间有什么区别？

6.  有哪些属性会传递给子进程？对于被 exec 执行的进程呢？
