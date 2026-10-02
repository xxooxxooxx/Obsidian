---
link-citations: true
title: "**CS341 系统编程课程手册**"
---

- [[#^post-mortems|事后复盘]]
  - [[#^pm-shellshock|ShellShock]]
  - [[#^pm-heartbleed|Heartbleed]]
  - [[#^pm-dirtycow|Dirty COW]]
  - [[#^pm-meltdown|Meltdown]]
  - [[#^pm-spectre|Spectre]]
  - [[#^pm-pathfinder|Mars Pathfinder]]
  - [[#^pm-mars-memory|再次火星]]
  - [[#^pm-y2038|2038 年问题]]
  - [[#^pm-blackout2003|2003 年北美东北部大停电]]
  - [[#^pm-ios-unicode|Apple iOS 的 Unicode 处理问题]]
  - [[#^pm-apple-ssl|Apple SSL 证书校验缺陷]]
  - [[#^pm-sony-rootkit|Sony Rootkit 安装事件]]
  - [[#^pm-steam-rm|Shell 脚本的惨痛教训]]
  - [[#^pm-appnexus|AppNexus 双重释放]]
  - [[#^pm-att1990|AT&T 连锁故障（1990）]]


# 事后复盘 ^post-mortems

**事后诸葛总是明明白白** — **佚名**

本章的目的，是回答一个更大的问题：“我们为什么要学这些东西？”在此前的所有课程中，你学的都是“该做什么”：如何实现一个数据结构、如何写一个 for 循环、如何证明某个命题。而这是第一门主要关注“*不*该做什么”的课程。因此，我们以真实的方式从过去汲取经验。坐下来慢慢翻完本章，听我们讲述过去那些程序员所遭遇的问题。即便你做的是 Web 开发这类层次高得多的工作，最终一切也都会追溯回系统本身。

## ShellShock ^pm-shellshock

先修：附录 / Shell

这是大多数 shell 中都存在的一个后门。这个 bug 允许攻击者利用环境变量来执行任意代码。

``` bash
$ env x='() { :;}; echo vulnerable' bash -c "echo this is a test"
vulnerable...
```

这意味着在任何使用环境变量、且不对其输入做净化的系统上（提示：没人会去净化环境变量输入，因为他们认为这是安全的），你都可以在别人的机器上执行任意代码，包括架设一个 Web 服务器。

经验教训：在生产机器上，要确保操作系统是最精简的（比如 BusyBox 配合 DietLibc），这样你才能理解系统中大部分代码及其有效性。多加几层抽象和检查，确保数据不会泄露。比如上面这个例子的问题恰恰在于：如果允许它与攻击者通信，信息就会被回传给攻击者。这意味着你可以通过只允许少数几个端口连接来加固机器端口。此外，你还可以加固系统，让它永远不通过 exec 调用来完成任务（即不要为了更新某个值就去做一次 exec 调用），而是在 C 语言或你最喜欢的编程语言里完成。虽然你失去了灵活性，但你可以安心地知道自己允许用户做什么。

## Heartbleed ^pm-heartbleed

先修：C 语言导论

简单来说，就是缓冲区检查没有边界。SSL 心跳机制极其简单：服务器发送一段特定长度的字符串，第二个服务器本应把该长度的字符串原样发回。问题在于，有人可以恶意地把请求的大小改得比实际发送的内容更大（比如只发“cat”，却请求 500 字节），从而从服务器拿到密码等关键信息。<a href="https://xkcd.com/1354/">https://xkcd.com/1354/</a> 这幅 xkcd 漫画讲的正是这件事。

经验教训：检查你的缓冲区！分清缓冲区与字符串的区别。

## Dirty COW ^pm-dirtycow

先修：进程 / 虚拟内存

<a href="https://en.wikipedia.org/wiki/Dirty_COW">https://en.wikipedia.org/wiki/Dirty_COW</a>

一个进程通常拥有一组只读的内存映射，如果尝试向它们写入就会触发段错误。Dirty COW 是这样一类漏洞：多个线程同时尝试访问同一块内存，指望其中一个线程能同时翻转 NX 位和可写位。之后，攻击者就可以修改这个页面。这个手法可以作用于 effective user id 位，于是该进程可以假装自己以 root 身份运行，并派生出一个 root shell，从而以普通 shell 的身份获得对系统的访问。

经验教训：内核里的自旋锁很难写。

## Meltdown ^pm-meltdown

Meltdown 是 Spectre 的近亲：两者都利用乱序执行来泄露程序本不该读取的数据。参见安全章节的第 <a href="#sec:spectre">#sec:spectre</a> 节。

## Spectre ^pm-spectre

参见安全章节的第 <a href="#sec:spectre">#sec:spectre</a> 节。

## Mars Pathfinder ^pm-pathfinder

先修章节：同步，以及一点调度

<a href="https://www.microsoft.com/en-us/research/people/mbj/#!just-for-fun">https://www.microsoft.com/en-us/research/people/mbj/#!just-for-fun</a>

火星探路者是 1997 年的一次任务，除了其他成果之外，它还在火星上收集了气象数据。着陆器使用一条单一的“信息总线”在各部件之间传递数据，而访问这条总线由一个互斥锁保护。整体架构相当简单：有一个高优先级的总线管理线程、一个中优先级的通信线程，以及一个低优先级的气象数据采集线程。调度器是抢占式的：每当一个更高优先级的任务就绪，它就会从任何低优先级任务手中夺走 CPU。

导致系统开始全面故障的模式是这样的：低优先级的数据采集线程获取了互斥锁，以便把它的数据发布到总线上。随后高优先级的总线线程试图获取同一个互斥锁，只能等待。接着中优先级的通信线程变为就绪，抢占了低优先级线程——**而此时低优先级线程仍持有那把互斥锁**。通信线程运行了很久，低优先级线程因此无法完成并释放互斥锁，而高优先级的总线线程就被它们两个同时堵住了。这被称为 `priority inversion`：一个中优先级的任务实际上阻止了一个高优先级任务运行。过了一段时间，看门狗定时器发现总线线程一直没有运行，于是重置了整个系统，每次都丢失数据。修复方案被上传到航天器上：为那把互斥锁开启 `priority inheritance`，这样当低优先级线程持有该锁时，它会临时以正在等待该锁的最高优先级线程的优先级运行。

经验教训：在你把锁和优先级混在一起之前，先理解优先级反转。如果一把锁被不同优先级的任务共享，那就为它开启优先级继承（或使用优先级上限）。把调试和追踪用的钩子留在部署后的系统里——工程师们正是靠这些钩子从地球上诊断并修复了这个问题的。

## 再次火星 ^pm-mars-memory

先修章节：Malloc

<a href="https://www.computerworld.com/article/2574759/data-storage-solutions/out-of-memory-problem-caused-mars-rover-s-glitch.html">https://www.computerworld.com/article/2574759/data-storage-solutions/out-of-memory-problem-caused-mars-rover-s-glitch.html</a>

短版本是：他们的内存用光了。长版本是：他们把内存、磁盘空间和交换空间全都用光了。这个故事的教训是？一定要写出能应对文件失败的代码，也要能应对文件被关闭、内存耗尽的情况，这样操作系统才能通过热交换文件来释放内存。另外要清理文件，假定你的临时目录大约只有总大小的百分之一或千分之一，并按此来规划。

## 2038 年问题 ^pm-y2038

先修章节：C 语言导论

<a href="https://en.wikipedia.org/wiki/Year_2038_problem">https://en.wikipedia.org/wiki/Year_2038_problem</a>

这是一个尚未发生的问题。Unix 时间戳以从某个特定日期（1970 年 1 月 1 日）起的秒数来保存，并以 32 位有符号整数存储。到 2038 年 3 月，这个数字就会溢出。对大多数存储 64 位有符号整数的现代操作系统来说这不是问题，那个容量足够我们用到时间的尽头；但对那些我们无法更改内部硬件的嵌入式设备来说，这就是个问题。敬请期待事态发展。

经验教训：按你的应用总有一天会变得极其庞大来规划。

## 2003 年北美东北部大停电 ^pm-blackout2003

先修章节：同步

<a href="https://en.wikipedia.org/wiki/Northeast_blackout_of_2003">https://en.wikipedia.org/wiki/Northeast_blackout_of_2003</a>

一个竞态条件在某个系统中触发了一连串未定义事件，导致北美东北部大部分地区停电了相当长一段时间。这个 bug 还关掉了、或导致备份系统和日志系统失效，以至于人们一个小时之内甚至都不知道出了这个 bug。具体是哪些位被翻转了并不清楚，但补丁已经发布了。

经验教训：把代码模块化，以便把故障局限在局部（比如让不同进程之间的竞态条件彼此独立）。如果你需要在进程之间做同步，请确保你的故障检测系统没有和主系统交织在一起。

## Apple iOS 的 Unicode 处理问题 ^pm-ios-unicode

先修章节：C 语言导论

<a href="http://appleinsider.com/articles/15/05/26/bug-in-ios-notifications-handling-crashes-iphones-with-a-simple-text">http://appleinsider.com/articles/15/05/26/bug-in-ios-notifications-handling-crashes-iphones-with-a-simple-text</a>

想知道我们为什么要教字符串解析吗？因为即便对专业软件开发者来说这也是件难事。这个 bug 使得在解析一系列 Unicode 字符时出现了大量未定义行为。Apple 大概知道这是怎么发生的，但我们猜测字符串的解析发生在内核内部的某处，结果遇到了段错误。当你在内核中遇到段错误时，内核会 panic，整台设备随之重启。未定义行为意味着任何事都可能发生，而这个 bug 确实引发了许多五花八门的情况。

经验教训：对你的内核做模糊测试（fuzz）。

## Apple SSL 证书校验缺陷 ^pm-apple-ssl

先修章节：C 语言导论

<a href="https://en.wikipedia.org/wiki/Unreachable_code#Examples">https://en.wikipedia.org/wiki/Unreachable_code#Examples</a>

由于 Apple 代码中一个多余的 goto，某个函数总会返回“SSL 证书有效”。自然地，黑客们得以用一些相当疯狂的名字来蒙混过关。

经验教训：if 语句永远加大括号，goto 尽量少用。很可能如果你真的需要用 goto，不如另写一个函数，或者用一个带 fall through 的 switch 语句（当然这样也还是不好）。

## Sony Rootkit 安装事件 ^pm-sony-rootkit

先修章节：C 语言导论 / 进程

<a href="https://en.wikipedia.org/wiki/Sony_BMG_copy_protection_rootkit_scandal">https://en.wikipedia.org/wiki/Sony_BMG_copy_protection_rootkit_scandal</a>

想象一下：那是 2005 年，Limewire 在几年前刚出现，互联网是一个正在不断滋生的非法活动池——别指望现在这问题已经解决了。Sony 清楚自己没有算力去管控整个互联网，也无法绕过人们用来规避版权保护的各种技术。于是他们做了什么？他们借助 2200 万张音乐 CD，强制用户在自己的操作系统上安装一个 rootkit，这样 Sony 就能监控设备上是否有不道德行为。

暂且不谈隐私方面的顾虑——相信我，顾虑非常多——真正的大问题是：如果这个 rootkit 编写不当，它就会成为所有人系统上的后门。rootkit 是一段通常安装在内核侧的代码，它会记录用户几乎所有的行为：能看到访问了哪些网站、点击了哪些按钮、输入了哪些按键等等。如果黑客得知了它的存在，并且存在从用户空间层级访问该 API 的途径，那就意味着任何程序都能获取关于你设备的重要信息。不用说，人们非常愤怒。

经验教训：装个杀毒软件和／或 apparmor，并确保一个应用只申请合理的权限。如果你拿不定主意，可以试试类似 Windows 沙箱的东西，或者留一台 sacrificial VM 看看安装它会不会把你的电脑搞得很难受。不要相信证书，要相信代码。

## Shell 脚本的惨痛教训 ^pm-steam-rm

先修章节：附录 / Shell

<a href="https://www.pcworld.com/article/2871653/scary-steam-for-linux-bug-erases-all-the-personal-files-on-your-pc.html">https://www.pcworld.com/article/2871653/scary-steam-for-linux-bug-erases-all-the-personal-files-on-your-pc.html</a>

Steam 里有一个简单的 bug，导致它把你所有的文件都删了，形式大概像这样

``` bash
STEAMROOT="$(cd "${0%/*}" && echo $PWD)"
# ...
rm -rf "$STEAMROOT/"*
```

在 shell 脚本中，$0 是脚本自身的路径（不是第一个参数，第一个参数是 $1），所以第一行试图找到脚本所在的目录。如果那个目录不存在会怎样？比如用户把它移动走了？那么 `cd` 会失败，`STEAMROOT` 就是空的，而第二行会删掉根目录下所有用户可写的东西。

经验教训：一定要做参数检查，永远永远永远都要对脚本做 `set -e`；如果你预期某条命令会失败，就显式地把它列出来。你也可以把 rm 别名成 mv，之后再清理垃圾。

## AppNexus 双重释放 ^pm-appnexus

先修章节：C 语言导论 / Malloc

<a href="https://medium.com/xandr-tech/2013-09-17-outage-postmortem-586b19ae4307">https://medium.com/xandr-tech/2013-09-17-outage-postmortem-586b19ae4307</a>
(<a href="https://web.archive.org/web/20230303210004/https://medium.com/xandr-tech/2013-09-17-outage-postmortem-586b19ae4307">https://web.archive.org/web/20230303210004/https://medium.com/xandr-tech/2013-09-17-outage-postmortem-586b19ae4307</a>)

AppNexus 使用一个异步垃圾回收器，当它认为对象不再被使用时，就回收堆中不同的部分。为了保持低延迟，当一个内存中的对象被删除时，它会先从其他对象那里解除链接，并把它的内存安排在未来某个安全的时刻释放——那时不可能还有线程在使用它。2013 年，一个极少被改动的对象被一次数据更新删除，而代码里的一个 bug 把该对象删除了两次。这次更新通过了校验，因为真正的释放当时还没发生，于是它被分发到了大约 900 台广告服务器上。当这些被排期的释放终于执行时，双重释放让它们几乎在同一时刻全部崩溃。

经验教训：非必要就别写这种凑合的代码。把系统模块化，设置内存上限，监控代码的不同部分并手工优化。不存在一个适合所有人的、万能兜底的垃圾回收器。即便是像 JVM 这样经过大量测试的实现，如果你想榨出它的性能，也需要一些额外的推动。

## AT&T 连锁故障（1990） ^pm-att1990

先修章节：C 语言导论

<a href="https://users.csc.calpoly.edu/~jdalbey/SWE/Papers/att_collapse.html">https://users.csc.calpoly.edu/~jdalbey/SWE/Papers/att_collapse.html</a>
(<a href="https://web.archive.org/web/20260610030535/http://users.csc.calpoly.edu/~jdalbey/SWE/Papers/att_collapse.html">https://web.archive.org/web/20260610030535/http://users.csc.calpoly.edu/~jdalbey/SWE/Papers/att_collapse.html</a>)

上面那个链接对这个 bug 解释得很清楚。我们建议读一读以了解更多。纽约的一台交换机在一次故障后自我重置，并告诉它的邻居自己已停止服务。当它重新上线时，它开始路由此前积压的呼叫，并向其他交换机发出一阵紧挨着的定时消息。由于 C 代码中有一处 `break` 语句写错了位置，一台在处理第一条消息时又收到第二条消息的交换机覆盖了自己的数据，并自我重置。当这些交换机中的每一台重新上线时，都会发出自己的一阵消息，于是重置在整个网络的全部 114 台交换机之间连锁扩散，持续了大约九个小时。

经验教训：一个放错位置的 `break` 就让一个国家的电话网络瘫痪了。务必清楚 C 语言里 `break` 的确切作用：退出的是最内层的 `switch` 或循环，绝不是一个 `if`。要测试你的失败路径和恢复路径，而不只是顺利路径，比如用模拟或模糊测试，以随机时序投递消息。恢复代码极少运行，所以它是你测试最少的代码，而当它同时在许多机器上出问题时，就能把一次故障变成一场雪崩。
