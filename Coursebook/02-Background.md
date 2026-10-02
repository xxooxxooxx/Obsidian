---
bibliography:
- background/background.bib
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^background|背景知识]]
  - [[#^systems-architecture|系统架构]]
    - [[#^assembly|汇编]]
    - [[#^atomic-operations|原子操作]]
    - [[#^caching|缓存]]
    - [[#^interrupts|中断]]
    - [[#^optional-hyperthreading|可选：超线程]]
  - [[#^debugging-and-environments|调试与开发环境]]
    - [[#^ssh|ssh]]
    - [[#^git|git]]
    - [[#^editors|编辑器]]
    - [[#^clean-code|整洁的代码]]
    - [[#^asserts|断言]]
  - [[#^valgrind|Valgrind]]
    - [[#^tsan|TSAN]]
  - [[#^gdb|GDB]]
    - [[#^involved-gdb-example|一个较复杂的 gdb 示例]]
    - [[#^shell|Shell]]
    - [[#^undefined-behavior-sanitizer|未定义行为检测器]]
    - [[#^clang-static-build-tools|Clang 静态构建工具]]
    - [[#^strace-and-ltrace|strace 与 ltrace]]
    - [[#^printfs|printfs]]
  - [[#^homework-0|作业 0]]
    - [[#^so-you-want-to-master-system-programming-and-get-a-better-grade-than-b|你想精通系统编程、并拿到比 B 更好的成绩吗？]]
    - [[#^watch-the-videos-and-write-up-your-answers-to-the-following-questions|观看视频并写下你对以下问题的回答]]
    - [[#^chapter-1|第 1 章]]
    - [[#^chapter-2|第 2 章]]
    - [[#^chapter-3|第 3 章]]
    - [[#^chapter-4|第 4 章]]
    - [[#^chapter-5|第 5 章]]
    - [[#^c-development|C 语言开发]]
    - [[#^optional-just-for-fun|可选：纯属娱乐]]
  - [[#^university-of-illinois-specific-guidelines|伊利诺伊大学专门指南]]
    - [[#^the-class-forum|课程论坛]]


# 背景知识 ^background

**有时候，千里之行的开端，是学会走路**——

## 系统架构 ^systems-architecture

本节简要回顾做系统编程所需的系统架构知识。

### 汇编 ^assembly

什么是汇编？汇编是你在不必亲手写 1 和 0 的情况下所能触及的机器语言最低层。每台计算机都有一种架构，而该架构又对应着一种汇编语言。每条汇编命令与一组 1 和 0 是一一对应的，这些 1 和 0 精确地告诉计算机该做什么。例如，在广泛使用的 x86 汇编语言中，下面这段代码把内存地址 `0x20` 处的字节加一
（<a href="#ref-wiki:xxx">[13]</a>）——你也可以查阅
（<a href="#ref-guide2011intel">[6]</a>）第 2A 节中 add 指令下的说明，不过那里写得比较啰嗦。

```
add BYTE PTR [0x20], 1
```

为什么要讲这些？因为尽管你在这门课中的大部分内容都将用 C 来写，但代码最终会被翻译成这样的形式，而这对竞态条件和原子操作有着重要影响。

### 原子操作 ^atomic-operations

如果一个操作不允许被其他处理器打断，那么它就是原子的。以刚才那段把某个内存地址加一的汇编代码为例。在该架构上，它在电路层面可能实际上有好几个不同的步骤。这个操作可能先从内存条中取出该内存地址的值，把它存入缓存或某个寄存器，最后再写回
（<a href="#ref-schweizer2015evaluating">[10]</a>）——这是在 *fetch-and-add* 的说明之下，不过你的
微架构可能有所不同。或者出于性能优化的考虑，它可能把该值保留在缓存中或某个对该进程而言是本地的寄存器里——试着导出一份对变量自增优化后的 `-O2` 汇编代码看看。问题出现在两个处理器试图同时做这件事的时候。两个处理器可能同时复制该内存地址的值、加上 1、再把相同的结果存回去，结果这个值只被加了一次。这就是为什么在现代系统上我们有一组称为原子操作的特殊指令。如果一条指令是原子的，它就确保同一时刻只有一个处理器或线程在执行任何中间步骤。在 x86 上，这通过 `lock` 前缀实现（<a href="#ref-guide2011intel">[6]</a>）。

    lock add BYTE PTR [0x20], 1

为什么我们不对所有东西都这么做？因为这会让命令变慢！如果计算机每次做点什么都得确认其他核心或处理器没在做事，那会慢得多。绝大多数时候我们会特意区分这些情况。也就是说，用到这类东西时我们会告诉你。绝大多数时候你可以假设这些指令是没有加锁的。

### 缓存 ^caching

啊，是的，缓存。计算机科学最棘手的问题之一。我们所说的缓存是处理器缓存。如果在读写时某个特定地址已经在缓存中，处理器就会在缓存上执行该操作（比如加法），稍后再更新真正的内存，因为更新内存很慢
（<a href="#ref-intel2015improving">[7]</a>）。如果
不在，处理器会向内存芯片请求一块内存并把它存入缓存，同时淘汰最久未使用的页——这取决于缓存策略，不过 Intel 的策略确实是这么做的。这样做是因为在时间上，处理器 L3 缓存的到达速度大约是内存的三倍
（<a href="#ref-levinthal2009performance">[9]</a>）
当然确切速度会随时钟频率和架构而变化。自然而然地，这会带来问题，因为同一个值存在两份不同的副本；在所引用的论文中，这被称为未共享的 cache line（unshared line）。这门课不是讲缓存的，但你应该知道它会如何影响你的代码。一份简短但不完整的清单如下：

1.  竞态条件！如果一个值被存放在两个不同的处理器缓存中，那么该值就应当只由一个线程访问。

2.  速度。有了缓存，你的程序可能会莫名其妙地变快。只要认为那些刚刚发生的读写、或者在内存中彼此相邻的读写是快的就行了。

3.  副作用。每次读或写都会影响缓存状态。多数时候这既没帮助也没损害，但知道这一点很重要。关于 lock 前缀的更多信息，请查阅 Intel 程序员手册。

### 中断 ^interrupts

中断是系统编程中重要的一部分。中断在内部是一个电信号，当某件事发生时会被送到处理器——这是一种硬件中断
（<a href="#ref-redhat_hardware_int">[3]</a>）。然后
硬件会判断这是否是它应该处理的事情（例如为老式键盘和鼠标处理键盘或鼠标输入），还是应该转交给操作系统。接着操作系统判断这是否是它应该处理的事情（例如把内存页从磁盘调入内存表），还是应该由应用程序来处理的事情（例如一次 SEGFAULT）。如果操作系统决定这应该由该进程或程序来应付，它就会发出一个**软件故障**，而这个软件故障随后被向上传播。应用程序再判断它是不是错误（SEGFAULT）还是不是（例如 SIGPIPE），并向用户报告。应用程序也可以向内核以及硬件发送信号。这是一种过度简化，因为确实存在某些无法忽略或屏蔽的硬件故障，但这门课不是要教你构建操作系统。

它的一个重要应用就是系统调用是如何被服务的！内核规定了一套成熟的寄存器，参数按其约定放入其中，还有一个同样由内核定义的系统调用"编号"。然后操作系统触发一个中断，内核捕获它并服务该系统调用
（<a href="#ref-garg_2006">[4]</a>）。

操作系统开发者和指令集开发者都不喜欢在系统调用上触发中断所带来的开销。如今，系统使用 `SYSENTER` 和 `SYSEXIT`，它们有一种更干净的方式，可以安全地把控制权转交给内核，再安全地转回来。至于"安全"具体意味着什么，显然超出了本课程的范围，但这个特性一直保留着。

### 可选：超线程 ^optional-hyperthreading

超线程是 Intel 对同时多线程（SMT）的称呼，这是一项 Intel 于 2002 年首次出货的硬件技术。尽管名字相似，它与软件多线程并不是一回事：一个物理核心对外呈现为两个共享其执行单元的逻辑 CPU。超线程让一个物理核心在操作系统看来像是许多虚拟核心
（<a href="#ref-guide2011intel">[6]</a>）。操作系统随后就可以在这些虚拟核心上调度进程，
而实际执行它们的是一个核心。每个核心在进程或线程之间交替切换。当核心在等待一次内存访问完成时，它可以去执行另一个进程线程的几条指令。总体结果是在更短时间内执行了更多指令。这可能意味着你可以用更少的核心数量来驱动较小的设备。

不过这里是有坑的。使用超线程时，你必须警惕各种优化。一个著名的超线程 bug 会导致程序崩溃，条件是至少有两个进程被调度到同一个物理核心上、在一个紧密循环中使用特定寄存器。真正的问题用架构的视角来解释更清楚。不过，实际的应用是系统程序员在维护 OCaml 主线时发现的
（<a href="#ref-leroy_2017">[8]</a>）。

## 调试与开发环境 ^debugging-and-environments

我要告诉你这门课的一个秘密：它讲的是如何更聪明地工作，而*不是*更努力地工作。这门课可能很耗时间，但这么多人这么认为（以及为什么这么多学生并不这么认为），原因在于人们对各自工具的熟悉程度不同。下面我们过一遍你会用到、也需要熟悉的一些常见工具。

### ssh ^ssh

`ssh` 是 Secure Shell 的缩写
（<a href="#ref-openbsd_ssh">[11]</a>）。它是一种网络
协议，允许你在远程机器上启动一个 shell。在这门课中，大多数时候你需要像这样 ssh 登录你的虚拟机：

``` bash
$ ssh netid@sem-cs341-VM.cs.illinois.edu
```

如果你不想每次都输入密码，可以生成一个 ssh 密钥来唯一标识你的机器。如果你已经有一对密钥，可以直接跳到复制 id 那一步。

``` bash
> ssh-keygen -t rsa -b 4096
# Do whatever keygen tells you
# Don't feel like you need a passcode if your login password is secure
> ssh-copy-id netid@sem-cs341-VM.cs.illinois.edu
# Enter your password for maybe the final time
> ssh  netid@sem-cs341-VM.cs.illinois.edu
```

如果你还是觉得这输入太多，你随时可以为各个主机设置别名。你可能需要重启虚拟机或重载 sshd 才能让配置生效。在 Linux 和 Mac 的发行版上可以直接使用配置文件。在 Windows 上，你得用 Windows Subsystem for Linux（WSL），或者在 PuTTY 中配置相应的别名。

``` bash
> cat ~/.ssh/config
Host vm
  User          netid
  HostName      sem-cs341-VM.cs.illinois.edu
> ssh vm
```

### git ^git

什么是"git"？git 是一个版本控制系统。它的意思是 git 会存储一个目录的全部历史。我们把这个目录称为一个仓库（repository）。所以你需要知道几件事。首先，用仓库创建工具建立你的仓库。如果你还没有登录学校的企业版 GitHub，一定要先登录，否则你的仓库不会被为你创建。之后，你的仓库就创建在服务器上了。git 是一个去中心化的版本控制系统，这意味着你需要把一个仓库弄到你的虚拟机上。我们可以这样做 clone。不管你做什么，**都不要跟着 README.md 的教程走**。把
`<semester>` 替换成当前学期的代码（例如 `fa26`），把
`<netid>` 替换成你的 NetID。

``` bash
$ git clone https://github.com/illinois-cs-coursework/<semester>_cs341_<netid>
```

这会创建一个本地仓库。工作流程是：你在本地仓库中做出修改，把改动加入当前的暂存区（add），真正地提交（commit），然后把改动推送到服务器。

``` bash
$ # edit the file, maybe using vim
$ git add <file>
$ git commit -m "Committing my file"
$ git push origin master
```

要把 git 讲清楚，你需要理解对我们而言 git 看起来就像一个链表。你永远位于 master 的头部，然后不断进行"编辑-添加-提交-推送"的循环。我们在 Github 上有一个单独的分支，会把反馈推送到那个特定分支下，你可以在 Github 网站上查看它。那个 markdown 文件里会有测试用例和结果的信息（比如标准输出）。

git 偶尔也会坏掉。下面是一份你可能用得上的、用来修复仓库的命令清单：

1.  git-cherry-pick

2.  git-pack

3.  git-gc

4.  git-clean

5.  git-rebase

6.  git-stash/git-apply/git-pop

7.  git-branch

通常，`git status` 会显示你当前在哪个分支上，输出类似
下面这样

``` bash
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

或者：

``` bash
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   <FILE>
        ...

no changes added to commit (use "git add" and/or "git commit -a")
```

如果你看到的却是下面这样，完全没有分支信息：

``` bash
$ git status
HEAD detached at 4bc4426
nothing to commit, working directory clean
```

不要慌，但你的仓库可能处于无法工作的状态。如果你不是快要截止日期了，来找我们当面问，或者在课程论坛上提问，我很乐意帮忙。在紧急情况下，删掉你的仓库然后重新 clone。**这会丢失任何尚未提交的本地改动。记得把你正在改动的文件复制到该目录之外，删除仓库之后再复制回来。**

如果你想更多地了解 git，网上有数不清的教程和资源可以帮到你。下面是几个可能有用的链接：

1.  <a href="https://git-scm.com/docs/gittutorial">https://git-scm.com/docs/gittutorial</a>

2.  <a href="https://www.atlassian.com/git/tutorials/what-is-version-control">https://www.atlassian.com/git/tutorials/what-is-version-control</a>

3.  <a href="https://thenewstack.io/tutorial-git-for-absolutely-everyone/">https://thenewstack.io/tutorial-git-for-absolutely-everyone/</a>

### 编辑器 ^editors

有些人把这里当作学习一个新编辑器的机会，另一些人则不那么在意。第一部分是说给你们中想学新编辑器的人听的。在那场持续数十年的编辑器大战中，我们来到了 vim 对 emacs 的战场。

vim 是一个文本编辑器，也是一个类 Unix 工具。你输入 `vim [file]` 进入 vim，这会把你带进编辑器。最常用的模式有三种：普通模式、插入模式和命令模式。你从普通模式开始。在这种模式下，你可以用很多按键移动，其中最常用的是 `hjkl`（分别对应左、下、上、右）。要在 vim 中执行命令，你可以先输入 `:`，然后在其后输入命令。比如要退出 vim，只需输入 `:q`（q 代表 quit）。如果你有未保存的编辑，要么用 `:w` 保存，要么用 `:wq` 保存并退出，要么用 `:q!` 退出并丢弃改动。要进行编辑，你可以输入 `i` 切换到插入模式，或输入 `a` 在光标**之后**切换到插入模式。这些就是 vim 的基本内容。除了网上数不清的优秀资源之外，vim 还为新手准备了内置教程。要访问交互式教程，在命令行（不是 vim 内部）输入 `vimtutor`，然后一切就绪！

emacs 更像是一种生活方式，而且我不是在打比方。很多人说 emacs 是一个 powerful 的操作系统，只是缺一个像样的文本编辑器。这意味着 emacs 可以容纳一个终端、一次 gdb 会话、一次 ssh 会话、代码以及大量别的东西。除了通过 gnu-docs <a href="https://www.gnu.org/software/emacs/tour/">https://www.gnu.org/software/emacs/tour/</a> 之外，用别的方式把你介绍给 gnu-emacs 都不太合适。
只需注意，emacs *极其*强大。你几乎可以用它做任何事。有相当多的学生喜欢其他编程语言在 IDE 方面的那种体验。请知道你可以把 emacs 配置成一个 IDE，但你得先学一点 Lisp
<a href="http://martinsosic.com/development/emacs/2017/12/09/emacs-cpp-ide.html">http://martinsosic.com/development/emacs/2017/12/09/emacs-cpp-ide.html</a>。

然后是那些喜欢用自己编辑器的人。这完全没有问题。为此我们需要 sshfs，它在许多不同机器上都有可用的移植版本。

1.  Windows
    <a href="https://github.com/billziss-gh/sshfs-win">https://github.com/billziss-gh/sshfs-win</a>

2.  Mac
    <a href="https://web.archive.org/web/20240913220726/https://github.com/osxfuse/osxfuse/wiki/SSHFS">https://web.archive.org/web/20240913220726/https://github.com/osxfuse/osxfuse/wiki/SSHFS</a>

3.  Linux
    <a href="https://help.ubuntu.com/community/SSHFS">https://help.ubuntu.com/community/SSHFS</a>

到这一步，你虚拟机上的文件就与你本机的文件同步了，你可以进行编辑，改动会被同步回去。

在写本文时，作者喜欢用 spacemacs
<a href="http://spacemacs.org/">http://spacemacs.org/</a>，它把
vim 和 emacs 两者以及两者各自的麻烦都撮合到了一起。我会在这里倾倒一下我为什么喜欢它，但要提醒你：如果你此前对 vim 或 emacs 完全没有经验，那么它的学习曲线连同这门课可能就太重了。

1.  可扩展。spacemacs 有一个用 lisp 写成、设计干净。只需编辑你的 spacemacs 配置并重新加载，就有数百个软件包可以安装，它们能做语法检查、自动静态分析等各种事情。

2.  vim 和 emacs 各自最好的部分。emacs 擅长靠成为一个快速编辑器把事情快速做完。vim 擅长快速编辑和快速移动。spacemacs 两全其美：让 vim 的按键绑定加上底下 emacs 的全部好处。

3.  预先配置了很多东西。与全新安装的 emacs 相比，很多与语言和项目相关的配置已经为你做好了，比如 neotree、helm 以及各种语言层。你要做的只是导航到项目的根目录，emacs 就会变成那个编程语言的 IDE。

不过显然还是各有所好。会有很多人争论说，编辑器的行家花在编辑自己编辑器上的时间比真正编辑还多。

### 整洁的代码 ^clean-code

用辅助函数让你的代码模块化。如果有一项重复的任务（比如在 malloc MP 中获取指向连续内存块的指针），就把它们做成辅助函数。并确保每个函数都只把一件事做好，这样你就不用调试两次。假设我们用每次迭代查找最小元素的方式来做选择排序：

``` objectivec
void selection_sort(int *a, long len){
     for(long i = len-1; i > 0; --i){
         long max_index = i;
         for(long j = len-1; j >= 0; --j){
             if(a[max_index] < a[j]){
                  max_index = j;
             }
         }
         int temp = a[i];
         a[i] = a[max_index];
         a[max_index] = temp;
     }

}
```

很多人能看出这段代码里的 bug，但把上面的方法重构成下面这些函数仍然是有帮助的：

``` objectivec
long max_index(int *a, long start, long end);
void swap(int *a, long idx1, long idx2);
void selection_sort(int *a, long len);
```

这样错误就明确地落在某一个函数里。归根结底，这门课讲的是编写系统程序，而不是一门关于重构/调试你代码的课。事实上，大多数内核代码糟糕到你根本不想读——为它辩护的理由是它确实得这样。但为了调试的缘故，采用其中一些做法从长远看可能对你有益。

### 断言 ^asserts

用断言来确保你的代码在某个阶段之前是正常的——更重要的是，确保你不会在之后把它弄坏。例如，如果你的数据结构是一个双向链表，你可以写点像
`assert(node == node->next->prev)` 这样的东西，来断言下一个节点有一个指回当前节点的指针。你也可以检查该指针是否指向一段符合预期的内存地址范围、是否非空、`-\>size` 是否合理，等等。定义 `NDEBUG` 宏（例如用 `-DNDEBUG` 编译）会禁用所有断言，所以等你调试完之后可以把它设上
（<a href="#ref-cplusplus_assert">[2]</a>）。

下面是一个带断言的简单例子。假设我们正在用 memcpy 写代码 我们想在它前面放一个断言，用来检查这两个内存区域是否重叠。如果它们确实重叠，memcpy 就会陷入未定义行为，所以我们希望更早而不是更晚发现这个问题。

``` objectivec
assert( src+n < dest || src >= dest + n); // source should finish before the destination or the source starts after the end of destination
memcpy(dest, src, n);
```

这个检查可以在编译期关掉，但它会帮你省下**大量**调试的麻烦！

## Valgrind ^valgrind

Valgrind 是一套工具，提供调试与性能分析手段，让你的程序更正确，并检测出一些运行时问题
（<a href="#ref-valgrind">[1]</a>）。其中最常用的是
Memcheck，它可以检测出许多与内存有关的错误——这些错误在 C 和 C++ 程序中很常见，并且可能导致崩溃和不可预测的行为（例如未释放的内存缓冲区）。要对你的程序运行 Valgrind：

``` objectivec
valgrind --leak-check=full --show-leak-kinds=all myprogram arg1 arg2
```

参数是可选的，默认运行的工具是 Memcheck。输出会以这种形式呈现：分配次数、释放次数和错误数。假设我们有一个像下面这样的简单程序：

``` objectivec
#include <stdlib.h>

void dummy_function() {
    int* x = malloc(10 * sizeof(int));
    x[10] = 0;        // error 1: Out of bounds write, as you can see here we write to an out of bound memory address.
}                    // error 2: Memory Leak, x is not freed at function exit.

int main(void) {
    dummy_function();
    return 0;
}
```

这个程序编译通过、运行也没有错误。我们来看看 Valgrind 会输出什么。

``` objectivec
==29515== Memcheck, a memory error detector
==29515== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==29515== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==29515== Command: ./a
==29515==
==29515== Invalid write of size 4
==29515==    at 0x108774: dummy_function (in /home/user/a)
==29515==    by 0x10878F: main (in /home/user/a)
==29515==  Address 0x4a7e068 is 0 bytes after a block of size 40 alloc'd
==29515==    at 0x4885250: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-arm64-linux.so)
==29515==    by 0x108767: dummy_function (in /home/user/a)
==29515==    by 0x10878F: main (in /home/user/a)
==29515==
==29515==
==29515== HEAP SUMMARY:
==29515==     in use at exit: 40 bytes in 1 blocks
==29515==   total heap usage: 1 allocs, 0 frees, 40 bytes allocated
==29515==
==29515== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==29515==    at 0x4885250: malloc (in /usr/libexec/valgrind/vgpreload_memcheck-arm64-linux.so)
==29515==    by 0x108767: dummy_function (in /home/user/a)
==29515==    by 0x10878F: main (in /home/user/a)
==29515==
==29515== LEAK SUMMARY:
==29515==    definitely lost: 40 bytes in 1 blocks
==29515==    indirectly lost: 0 bytes in 0 blocks
==29515==      possibly lost: 0 bytes in 0 blocks
==29515==    still reachable: 0 bytes in 0 blocks
==29515==         suppressed: 0 bytes in 0 blocks
==29515==
==29515== For lists of detected and suppressed errors, rerun with: -s
==29515== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

**Invalid write**：它检测到了我们的堆块越界写，也就是在已分配块的外部写入。

**Definitely lost**：内存泄漏——你大概忘了释放某个内存块。

Valgrind 是一个检查运行时错误的有效工具。C 在这类行为上比较特殊，所以在你编译完程序之后，可以用 Valgrind 修掉编译器可能漏掉、而通常在程序运行时才暴露出来的错误。

更多信息可以查阅手册
（<a href="#ref-valgrind">[1]</a>）。

### TSAN ^tsan

ThreadSanitizer 是 Google 的一个工具，内置于 clang 和 gcc 中，帮助你检测代码里的竞态条件
（<a href="#ref-threadsanitizercppmanual_2018">[12]</a>）。
注意，用 tsan 运行会让你的代码稍微变慢。考虑下面这段代码。

``` objectivec
#include <pthread.h>
#include <stdio.h>

int global;

void *Thread1(void *x) {
    global++;
    return NULL;
}

int main() {
    pthread_t t[2];
    pthread_create(&t[0], NULL, Thread1, NULL);
    global = 100;
    pthread_join(t[0], NULL);
}
// compile with gcc -fsanitize=thread -pie -fPIC -ltsan -g simple_race.c
```

我们可以看到变量 `global` 上存在竞态条件。主线程和新建的线程会同时试图修改该处的值。但是，ThreadSanitizer 能发现它吗？

``` bash
$ ./a.out
==================
WARNING: ThreadSanitizer: data race (pid=28888)
  Read of size 4 at 0x7f73ed91c078 by thread T1:
    #0 Thread1 /home/zmick2/simple_race.c:7 (exe+0x000000000a50)
    #1  :0 (libtsan.so.0+0x00000001b459)

  Previous write of size 4 at 0x7f73ed91c078 by main thread:
    #0 main /home/zmick2/simple_race.c:14 (exe+0x000000000ac8)

  Thread T1 (tid=28889, running) created by main thread at:
    #0  :0 (libtsan.so.0+0x00000001f6ab)
    #1 main /home/zmick2/simple_race.c:13 (exe+0x000000000ab8)

SUMMARY: ThreadSanitizer: data race /home/zmick2/simple_race.c:7 Thread1
==================
ThreadSanitizer: reported 1 warnings
```

如果我们用 debug 标志编译，它还会把变量名一并告诉我们。

## GDB ^gdb

GDB 是 GNU Debugger 的缩写。GDB 是一个程序，它通过交互式调试帮助你追踪错误
（<a href="#ref-gdb">[5]</a>）。它可以启动和停止你的程序、
四处查看，并临时加入各种约束和检查。下面是几个例子。

##### 以编程方式设置断点

断点是你希望执行停下来、把控制权交回调试器的那一行代码。用 GDB 调试复杂 C 程序时，一个有用的技巧是在源代码中设置断点。

``` objectivec
int main() {
    int val = 1;
    val = 42;
    asm("int $3"); // set a breakpoint here
    val = 7;
}
```

``` bash
$ gcc main.c -g -o main
$ gdb --args ./main
(gdb) r
[...]
Program received signal SIGTRAP, Trace/breakpoint trap.
main () at main.c:5
5     val = 7;
(gdb) p val
$1 = 42
```

你也可以在 gdb 内部交互式地设置断点。假设我们没有优化，行号如下：

``` bash
1. int main() {
2.     int val = 1;
3.     val = 42;
4.     val = 7;
5. }
```

现在我们可以在程序启动前设置断点了。

``` bash
$ gcc main.c -g -o main
$ gdb --args ./main
(gdb) break main.c:4
[...]
(gdb) r
[...]
(gdb) p val
$1 = 42
```

##### 检查内存内容

我们也可以用 gdb 检查不同内存片段的内容。例如，考虑这个程序：

``` objectivec
int main() {
    char bad_string[3] = {'C', 'a', 't'};
    printf("%s", bad_string);
}
```

编译并运行后，我们得到：

``` bash
$ gcc main.c -g -o main && ./main
$ Cat ZVQ� $
```

现在我们可以用 gdb 查看该字符串的特定字节，并推断程序本应在哪里停下：

``` bash
(gdb) l
1 #include <stdio.h>
2 int main() {
3     char bad_string[3] = {'C', 'a', 't'};
4     printf("%s", bad_string);
5 }
(gdb) b 4
Breakpoint 1 at 0x100000f57: file main.c, line 4.
(gdb) r
[...]
Breakpoint 1, main () at main.c:4
4     printf("%s", bad_string);
(gdb) x/16xb bad_string
0x7fff5fbff9cd: 0x63  0x61  0x74  0xe0  0xf9  0xbf  0x5f  0xff
0x7fff5fbff9d5: 0x7f  0x00  0x00  0xfd  0xb5  0x23  0x89  0xff
(gdb)
```

在这里，通过带参数 `16xb` 使用 `x` 命令，我们能看到从内存地址 `0x7fff5fbff9cd`（值为 `bad_string`）
开始，`printf` 实际上会看到如下的字节序列被当作字符串，
因为我们提供的是一个缺少结尾 null 的畸形字符串。

### 一个较复杂的 gdb 示例 ^involved-gdb-example

下面就是你的某位助教会如何调试一个出问题的简单程序。首先是程序的源代码。如果你能立刻看出错误，请给我们一点耐心。

``` objectivec
#include <stdio.h>

double convert_to_radians(int deg);

int main(){
    for (int deg = 0; deg > 360; ++deg){
        double radians = convert_to_radians(deg);
        printf("%d. %f\n", deg, radians);
    }
    return 0;
}

double convert_to_radians(int deg){
    return ( 31415 / 1000 ) * deg / 180;
}
```

我们该怎么用 gdb 来调试？首先我们得加载 GDB。

``` bash
$ gdb --args ./main
(gdb) layout src; # If you want a GUI type
(gdb) run
(gdb)
```

想看看源代码吗？

``` bash
(gdb) l
1   #include <stdio.h>
2   
3   double convert_to_radians(int deg);
4   
5   int main(){
6       for (int deg = 0; deg > 360; ++deg){
7           double radians = convert_to_radians(deg);
8           printf("%d. %f\n", deg, radians);
9       }
10      return 0;
(gdb) break 7 # break <file>:line or break <file>:function
(gdb) run
(gdb)
```

从运行结果看，断点甚至都没触发，说明代码根本没执行到那一行。这是由那个比较导致的！好吧，把符号翻转过来，现在应该可以了吧？

``` bash
(gdb) run
350. 60.000000
351. 60.000000
352. 60.000000
353. 60.000000
354. 60.000000
355. 61.000000
356. 61.000000
357. 61.000000
358. 61.000000
359. 61.000000
```

``` bash
(gdb) break 14 if deg == 359 # Let's check the last iteration only
(gdb) run
...
(gdb) print/x deg # print the hex value of degree
$1 = 0x167
(gdb) print (31415/1000)
$2 = 31
(gdb) print (31415/1000.0)
$3 = 31.414999999999999
(gdb) print (31415.0/10000.0)
$4 = 3.1415000000000002
```

不过那只是最起码的东西，你们大多数人靠这些也够了。网上还有大量资源，下面是几个能帮你入门的具体资源。

1.  <a href="http://www.cs.cmu.edu/~gilpin/tutorial/">http://www.cs.cmu.edu/~gilpin/tutorial/</a>

2.  <a href="https://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_toc.html">https://ftp.gnu.org/old-gnu/Manuals/gdb/html_node/gdb_toc.html</a>

3.  <a href="https://www.youtube.com/watch?v=PorfLSr3DDI">https://www.youtube.com/watch?v=PorfLSr3DDI</a>

### Shell ^shell

你到底用什么来运行你的程序？shell！shell 是一种运行在你终端里的编程语言。终端不过是一个输入命令的窗口。在 POSIX 上我们通常有一个叫 `sh` 的 shell，它链接到一个符合 POSIX 的 shell `dash`。大多数时候你用的是叫 `bash` 的 shell，它在某种程度上符合 POSIX，但有一些好用的内置特性。如果你想更进一步，`zsh` 还有一些更强大的特性，比如对程序名做 tab 补全以及模糊匹配。

### 未定义行为检测器 ^undefined-behavior-sanitizer

未定义行为检测器是 llvm 项目提供的一个很棒的工具。它让你能够用一个运行时检查器来编译代码，以确保你不会在各个类别上产生未定义行为。我们会尽量把它纳入我们的项目中，但它需要我们使用的所有外部库都提供支持，所以我们可能没法全部覆盖。
<a href="https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html">https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html</a>

#### 未定义行为——为什么我们无法普遍地解决它

另外，请务必读一读 Chris Lattner 那篇关于未定义行为的三部曲博文。它能让你看清调试构建以及编译器优化之谜。

<a href="http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html">http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html</a>

### Clang 静态构建工具 ^clang-static-build-tools

Clang 提供了很棒的、可直接替换的编译工具。
如果你想看看是否存在可能导致竞态条件的错误、转型错误等等，你只需要做下面这件事。

``` bash
$ scan-build make
```

除了 make 的输出之外，你还会得到静态构建的警告。

### strace 与 ltrace ^strace-and-ltrace

strace 和 ltrace 是两个程序，分别用于跟踪运行中程序或命令的系统调用和库调用。你的系统上可能没装它们，所以要安装的话运行下面这条命令：

``` bash
$ sudo apt install strace ltrace
```

用 ltrace 调试可以简单到只需弄清楚上一个失败的库调用的返回值是什么。

``` objectivec
int main() {
    FILE *fp = fopen("I don't exist", "r");
    fprintf(fp, "a");
    fclose(fp);
    return 0;
}
```

``` bash
> ltrace ./a.out
__libc_start_main(0xaaaaab1a0818, 1, 0xffffc0fdaa28, 0 <unfinished ...>
fopen("I don't exist", "r")                      = 0
fputc('a', 0 <no return ...>
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
```

注意编译器已经把单字符的 `fprintf` 替换成了对 `fputc` 的调用，而它得到的 `FILE*` 是 `0`（`NULL`），也就是 `fopen` 返回的那个值。ltrace 的输出能让你发现程序运行时正在做的一些奇怪事情。遗憾的是，ltrace 无法用来注入故障，也就是说 ltrace 能告诉你正在发生什么，却无法篡改已经在发生的事情。

另一方面，strace 就能修改你的程序。用 strace 调试非常棒。基本用法是对一个程序运行 strace，它会给你一份完整的系统调用参数列表。

``` bash
$ strace head README.md
execve("/usr/bin/head", ["head", "README.md"], 0x7ffff28c8fa8 /* 60 vars */) = 0
brk(NULL)                               = 0x7ffff5719000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=32804, ...}) = 0
...
```

如果输出太冗长，你可以用 trace= 加一个逗号分隔的系统调用列表来过滤掉除这些调用之外的所有内容。

``` bash
$ strace -e trace=read,write head README.md
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\260\34\2\0\0\0\0\0"..., 832) = 832
read(3, "# Locale name alias data base.\n#"..., 4096) = 2995
read(3, "", 4096)                       = 0
read(3, "# C Datastructures\n\n[![Build Sta"..., 8192) = 1250
write(1, "# C Datastructures\n", 19# C Datastructures
```

你也可以跟踪文件或目标。

``` bash
$ strace -e trace=read,write -P README.md head README.md
strace: Requested path 'README.md' resolved into '/mnt/c/Users/user/personal/libds/README.md'
read(3, "# C Datastructures\n\n[![Build Sta"..., 8192) = 1250
```

较新版本的 strace 实际上可以用 `-e inject=` 选项向你的程序注入故障。例如，
`strace -e inject=read:error=EIO head README.md` 会让每一次 `read` 调用
以 `EIO` 失败。当你想偶尔让读写失败时这很有用，比如在一个网络应用中，而你的程序本应能处理这种情况。当前 Ubuntu 发行版中的 strace 支持故障注入，所以上面安装的那个包就够了。

### printfs ^printfs

当其他一切都不管用时，就打印！你的每个函数都应该清楚自己打算做什么。你想测试每个函数是否在做它该做的事，并确切地看到代码在哪里崩掉。在竞态条件的情况下，tsan 可能帮得上忙，但让每个线程在特定时刻把数据打印出来也能帮你定位竞态条件。

为了让 printf 有用，尽量搞一个宏来填入调用 printf 时的上下文——可以说就是一条日志语句。一条简单、有用但未经测试的日志语句可以是这样：先写个测试、找出出问题的所在，然后把变量的状态记录下来。

``` objectivec
#include <execinfo.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <stdarg.h>
  #include <unistd.h>

  // bt is print backtrace
  const int num_stack = 10;
  int __log(int line, const char *file, int bt, const char *fmt, ...) {
    if (bt) {
      void *raw_trace[num_stack];
      size_t size = backtrace(raw_trace, sizeof(raw_trace) / sizeof(raw_trace[0]));
      char **syms = backtrace_symbols(raw_trace, size);

      for(ssize_t i = 0; i < size; i++) {
        fprintf(stderr, "|%s:%d| %s\n", file, line, syms[i]);
      }
      free(syms);
    }
    int ret = fprintf(stderr, "|%s:%d| ", file, line);
    va_list args;
    va_start(args, fmt);
    ret += vfprintf(stderr, fmt, args);
    va_end(args);
    ret += fprintf(stderr, "\n");
    return ret;
  }

  #ifdef DEBUG
  #define log(...) __log(__LINE__, __FILE__, 0, __VA_ARGS__)
  #define bt(...) __log(__LINE__, __FILE__, 1, __VA_ARGS__)
  #else
  #define log(...)
  #define bt(...)
  #endif

  //Use as log(args like printf) or bt(args like printf) to either log or get backtrace

  int main() {
    log("Hello Log");
    bt("Hello Backtrace");
  }
```

然后按需使用。如果你不清楚一个 C 程序是如何被翻译成机器码的，去看看附录中的编译与链接一节。

## 作业 0 ^homework-0

``` objectivec
// First, can you guess which lyrics have been transformed into this C-like system code?
char q[] = "Do you wanna build a C99 program?";
#define or "go debugging with gdb?"
static unsigned int i = sizeof(or) != strlen(or);
char* ptr = "lathe";
size_t come = fprintf(stdout,"%s door", ptr+2);
int away = ! (int) * "";

int* shared = mmap(NULL, sizeof(int*), PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
munmap(shared,sizeof(int*));

if(!fork()) {
    execlp("man","man","-3","ftell", (char*)0); perror("failed");
}

if(!fork()) {
    execlp("make","make", "snowman", (char*)0); execlp("make","make", (char*)0);
}

exit(0);
```

### 你想精通系统编程、并拿到比 B 更好的成绩吗？ ^so-you-want-to-master-system-programming-and-get-a-better-grade-than-b

``` C
int main(int argc, char** argv) {
    puts("Great! We have plenty of useful resources for you, but it's up to you to");
    puts(" be an active learner and learn how to solve problems and debug code.");
    puts("Bring your near-completed answers to the problems below");
    puts(" to the first lab to show that you've been working on this.");
    printf("A few \"don't knows\" or \"unsure\" is fine for lab 1.\n");
    puts("Warning: you and your peers will work hard in this class.");
    puts("This is not CS225; you will be pushed much harder to");
    puts(" work things out on your own.");
    fprintf(stdout,"This homework is a stepping stone to all future assignments.\n");
    char p[] = "So, you will want to clear up any confusions or misconceptions.\n";
    write(1, p, strlen(p) );
    char buffer[1024];
    sprintf(buffer,"For grading purposes, this homework 0 will be graded as part of your lab %d work.\n", 1);
    write(1, buffer, strlen(buffer));
    printf("Press Return to continue\n");
    read(0, buffer, sizeof(buffer));
    return 0;
}
```

### 观看视频并写下你对以下问题的回答 ^watch-the-videos-and-write-up-your-answers-to-the-following-questions

**重要！**

HW0 所需的浏览器内虚拟机和视频在这里：

<a href="http://cs-education.github.io/sys/">http://cs-education.github.io/sys/</a>

有问题？有想法？使用当前学期的 CS 341 Ed 讨论区，入口链接来自
<a href="https://cs341.cs.illinois.edu/">https://cs341.cs.illinois.edu/</a>。

浏览器内的虚拟机完全用 JavaScript 运行，在 Chrome 中速度最快。请注意，重新加载页面时该虚拟机以及你写的任何代码都会被重置，**所以要把你的代码复制到一个单独的文档里。** 视频后的小挑战不属于作业 0 的内容，但动手做比被动观看能学到更多。每个视频末尾的挑战都好好玩玩。

下面是 HW0 的问题。把你的答案复制到一个文本文档里，因为课程后面你需要提交它们。

### 第 1 章 ^chapter-1

在这一章里，我们英勇的主人公与标准输出、标准错误、文件描述符以及向文件写入展开搏斗。

1.  **Hello, World!（系统调用风格）** 写一个程序，用
    `write()` 打印出 "Hi! My name is \<Your Name\>"。

2.  **Hello, Standard Error Stream!** 写一个函数，向标准错误打印一个高度为 `n` 的三角形。你的函数应当具有
    签名 `void write_triangle(int n)`，并且应当使用 `write()`。
    当 n = 3 时，三角形应长这样：

    ``` objectivec
    *
    **
    ***
    ```

3.  **向文件写入** 拿你 "Hello, World!" 中的程序
    修改成写入一个叫 `hello_world.txt` 的文件。确保为 `open()` 使用正确的标志位和正确的模式（`man 2 open`
    是你的朋友）。

4.  **并非一切都是系统调用** 拿你 "向文件写入" 中的程序，
    把 `write()` 替换成 `printf()`。*一定要打印到文件，而不是标准输出！*

5.  `write()` 和 `printf()` 有什么差别？

### 第 2 章 ^chapter-2

衡量 C 类型及其取值范围、`int` 与 `char` 数组，以及指针自增。

1.  一个字节有多少位？

2.  一个 `char` 有多少个字节？

3.  以下类型在你的机器上各占多少字节？`int`、`double`、
    `float`、`long` 和 `long long`

4.  在一个整数为 8 字节的机器上，变量
    `data` 的声明是 `int data[8]`。如果 data 的地址是 `0x7fbd9d40`，
    那么 `data+2` 的地址是什么？

5.  `data[3]` 在 C 中等价于什么？提示：C 在解引用该地址之前会把
    `data[3]` 转换成什么？记住，字符串常量 `"abc"` 的类型是数组。

6.  这段代码为什么会 SEGFAULT？

    ``` objectivec
    char *ptr = "hello";
    *ptr = 'J';
    ```

7.  变量 `str_size` 的值是多少？

    ``` objectivec
    ssize_t str_size = sizeof("Hello\0World");
    ```

8.  变量 `str_len` 的值是多少

    ``` objectivec
    ssize_t str_len = strlen("Hello\0World");
    ```

9.  举一个 X 的例子，使得 `sizeof(X)` 等于 3。

10. 举一个 Y 的例子，使得 `sizeof(Y)` 依机器不同
    可能是 4 或 8。

### 第 3 章 ^chapter-3

程序参数、环境变量，以及字符数组（字符串）的使用。

1.  至少有哪两种方法可以求出 `argv` 的长度？

2.  `argv[0]` 表示什么？

3.  指向环境变量的指针存放在哪里（栈上、堆上、还是别处）？

4.  在一个指针为 8 字节的机器上，给出以下代码：

    ``` objectivec
    char *ptr = "Hello";
    char array[] = "Hello";
    ```

    What are the values of `sizeof(ptr)` and `sizeof(array)`? Why?

5.  什么数据结构管理着自动变量的生命周期？

### 第 4 章 ^chapter-4

堆和栈内存，以及结构体的使用。

1.  如果我想在创建它的函数生命周期结束之后继续使用数据，我应该把它放在哪里？我该怎么放进去？

2.  堆内存与栈内存有哪些差别？

3.  一个进程中还有其他种类的内存吗？

4.  填空："在一个好的 C 程序中，每一次 malloc 都对应一个
    \_\_\_"。

5.  `malloc` 可能失败的一个原因是什么？

6.  `time()` 和 `ctime()` 有什么差别？

7.  这段代码片段有什么问题？

    ``` objectivec
    free(ptr);
    free(ptr);
    ```

8.  这段代码片段有什么问题？

    ``` objectivec
    free(ptr);
    printf("%s\n", ptr);
    ```

9.  怎样可以避免前面这两个错误？

10. 创建一个代表 `Person` 的 `struct`。然后再做一个 `typedef`，
    这样 `struct Person` 就可以被替换成一个单词了。一个 Person 应当包含
    以下信息：他们的名字（一个字符串）、
    他们的年龄（一个整数），以及他们朋友的一份列表（存为一个
    指向 `Person` 指针数组的指针）。

11. 现在，在堆上创建两个 Person——"Agent Smith" 和 "Sonny Moore"，
    他们分别为 128 岁和 256 岁，并且互为朋友。创建函数来创建和销毁
    一个 Person（Person 及其名字应当存活在堆上）。

12. `create()` 应当接受一个名字和年龄。名字应当被复制到堆上。用 malloc
    为每个 Person 最多十个朋友预留足够的内存。记得初始化所有字段（为什么？）。

13. `destroy()` 应当同时释放 person 结构体的内存，以及存放在堆上的
    它的所有属性。销毁一个人时其他人保持完好。

### 第 5 章 ^chapter-5

文本输入输出，以及使用 `getchar`、`gets` 和
    `getline` 进行解析。

1.  哪些函数可以用来从 `stdin` 读取字符并
    把它们写到 `stdout`？

2.  说出 `gets()` 的一个问题。

3.  写代码解析字符串 "Hello 5 World"，并把 3 个
    变量分别初始化为 "Hello"、5 和 "World"。

4.  在包含 `getline()` 之前需要先定义什么？

5.  写一个 C 程序，用 `getline()`
    逐行打印一个文件的内容。

### C 语言开发 ^c-development

下面是关于使用编译器和 git 进行编译与开发的一些通用提示。这里做些网页搜索会有帮助。

1.  用哪个编译器标志来生成调试构建？

2.  你在 Makefile 中修好了一个问题，然后再次输入 `make`。解释一下为什么这可能不足以生成一次新的构建。

3.  在 Makefile 中规则之后的命令是用制表符还是空格缩进的？

4.  `git commit` 做什么？在 git 的语境下 `sha` 是什么？

5.  `git log` 会向你展示什么？

6.  `git status` 会告诉你什么？`.gitignore` 的内容会如何改变
    它的输出？

7.  `git push` 做什么？为什么只用它来提交是不够的，
    而要用 `git commit -m 'fixed all bugs' `？

8.  non-fast-forward 错误 `git push` reject 是什么意思？处理它最常用的
    方法是什么？

### 可选：纯属娱乐 ^optional-just-for-fun

- 把歌词改写成系统编程和本 wiki 书中涉及的 C 代码，并分享到课程论坛上。

- 凭你的眼光，找出网上最好的和最差的 C 代码，把链接发到课程论坛上。

- 写一个带有一个刻意设置的微妙 C bug 的小程序，发到课程论坛上，看看别人能不能找出你的 bug。

- 你听说过什么很酷/很灾难性的系统编程 bug 吗？欢迎在课程论坛上与你的同学和课程助教分享。

## 伊利诺伊大学专门指南 ^university-of-illinois-specific-guidelines

### 课程论坛 ^the-class-forum

助教和学生助理会收到海量的提问。有些问题做过充分调研，有些没有。下面这份实用指南能帮助你从前者走向后者。哦对了，我有没有提到过，这也是给实习经理留下好印象的一个轻松办法？问问你自己……

1.  我是在自己的虚拟机上操作吗？

2.  **我查过 man 手册了吗？**

3.  我在课程论坛上搜过类似的问题/后续讨论了吗？

4.  我把 MP/Lab 的说明完整读完了吗？

5.  我把所有视频都看完了吗？

6.  必要时，我 Google 过那条报错信息以及它的几种变体了吗？StackOverflow 呢。

7.  我试过把部分代码逐行注释掉、打印出来，或者单步执行，以精确定位错误发生的位置吗？

8.  **我把代码提交到 git 了吗，以便助教需要更多上下文时查看？**

9.  我在课程论坛的帖子中同时附上了控制台/GDB/Valgrind 的输出**以及**bug 周围的代码吗？

10. 我是否修好了与我当前问题无关的其他段错误？

11. 我是否遵循了良好的编程实践？（即封装、用函数来减少重复等等）

如果你想在课程论坛上得到迅速答复，我们能给你的最大建议是：**像你在尝试回答这个问题一样去提问**。也就是说，在提问之前，先试着自己回答它。如果你正打算发这样的帖子：

> 你好，我的代码在自动评分器上得了 50%。我反复测试过它，但
> 怎么都让它不出错。你能给我一些关于测试
> 用例的提示吗？

这样写没问题，也很有礼貌，但课程团队更希望看到类似下面这样的帖子：

> 你好，我最近在测试 X、Y、Z 上失败了，这大约是
> 这次作业一半的测试。我注意到它们都和
> 网络以及 epoll 有关，但没能弄清楚是什么把它们
> 联系在一起，也可能是我完全搞错了方向。所以为了验证我的想法，
> 我试着启动了 1000 个客户端，发出各种 get 和 put 请求，并
> 核对文件与原始文件是否一致。我在正常运行、
> 调试构建、valgrind 或 tsan 下都没能让它失败。我没有任何
> 警告，语法预检查也没有向我显示任何东西。
> 你能否告诉我我对这个失败原因的理解是否正确，以及
> 我可以怎样修改我的测试以更好地反映 X、Y、Z？netid：
> bvenkat2

你不必写得这么客气——虽然我们会很感激——但这样会明显更快地得到回复。如果你是在尝试回答这个问题，你会觉得所有需要的信息都已经在问题正文里了。

<div id="refs" class="references csl-bib-body hanging-indent">

<div id="ref-valgrind" class="csl-entry">

“4. Memcheck: A Memory Error Detector.” n.d. In *Valgrind*.
<a href="http://valgrind.org/docs/manual/mc-manual.html">http://valgrind.org/docs/manual/mc-manual.html</a>.

</div>

<div id="ref-cplusplus_assert" class="csl-entry">

“Assert.” n.d. In *Cplusplus.com*. Cplusplus.com.
<a href="http://www.cplusplus.com/reference/cassert/assert/">http://www.cplusplus.com/reference/cassert/assert/</a>.

</div>

<div id="ref-redhat_hardware_int" class="csl-entry">

“Chapter 3. Hardware Interrupts.” n.d. In *Chapter 3. Hardware
Interrupts*. Red Hat.
<a href="https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_MRG/1.3/html/Realtime_Reference_Guide/chap-Realtime_Reference_Guide-Hardware_interrupts.html">https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_MRG/1.3/html/Realtime_Reference_Guide/chap-Realtime_Reference_Guide-Hardware_interrupts.html</a>.

</div>

<div id="ref-garg_2006" class="csl-entry">

Garg, Manu. 2006. “Sysenter Based System Call Mechanism in Linux 2.6.”
In *Manu’s Public Articles and Projects*.
<a href="http://articles.manugarg.com/systemcallinlinux2_6.html">http://articles.manugarg.com/systemcallinlinux2_6.html</a>.

</div>

<div id="ref-gdb" class="csl-entry">

“GDB: The GNU Project Debugger.” 2019. In *GDB: The GNU Project
Debugger*. Free Software Foundation.
<a href="https://www.gnu.org/software/gdb/">https://www.gnu.org/software/gdb/</a>.

</div>

<div id="ref-guide2011intel" class="csl-entry">

Guide, Part. 2011. “Intel® 64 and Ia-32 Architectures Software
Developer’s Manual.” *Volume 3B: System Programming Guide, Part* 2.

</div>

<div id="ref-intel2015improving" class="csl-entry">

Intel, CAT. 2015. “Improving Real-Time Performance by Utilizing Cache
Allocation Technology.” *Intel Corporation, April*.

</div>

<div id="ref-leroy_2017" class="csl-entry">

Leroy, Xavier. 2017. “How i Found a Bug in Intel Skylake Processors.” In
*How I Found a Bug in Intel Skylake Processors*.
<a href="http://gallium.inria.fr/blog/intel-skylake-bug/">http://gallium.inria.fr/blog/intel-skylake-bug/</a>.

</div>

<div id="ref-levinthal2009performance" class="csl-entry">

Levinthal, David. 2009. “Performance Analysis Guide for Intel Core I7
Processor and Intel Xeon 5500 Processors.” *Intel Performance Analysis
Guide* 30: 18.

</div>

<div id="ref-schweizer2015evaluating" class="csl-entry">

Schweizer, Hermann, Maciej Besta, and Torsten Hoefler. 2015. “Evaluating
the Cost of Atomic Operations on Modern Architectures.” *2015
International Conference on Parallel Architecture and Compilation
(PACT)*, 445–56.

</div>

<div id="ref-openbsd_ssh" class="csl-entry">

“Ssh(1).” n.d. In *OpenBSD Manual Pages*. OpenBSD.
<a href="https://man.openbsd.org/ssh.1">https://man.openbsd.org/ssh.1</a>.

</div>

<div id="ref-threadsanitizercppmanual_2018" class="csl-entry">

“ThreadSanitizerCppManual.” 2018. In *ThreadSanitizerCppManual*. Google.
<a href="https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual">https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual</a>.

</div>

<div id="ref-wiki:xxx" class="csl-entry">

Wikibooks. 2018. *X86 Assembly — Wikibooks, the Free Textbook Project*.
<a href="https://en.wikibooks.org/w/index.php?title=X86_Assembly&oldid=3477563">https://en.wikibooks.org/w/index.php?title=X86_Assembly&oldid=3477563</a>.

</div>

</div>
