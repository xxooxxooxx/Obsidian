---
title: "Makefile-常用函数和通用模板"
source: "https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/#1-makefile-gui-ze"
author:
  - "[[John Doe]]"
published: 2024-06-24
created: 2026-10-03
description: "1 Makefile规则 2 一步一步完善 Makefile 2.1 模式规则 2.2 自动变量 2.2.1 函数传参   2.3 立即变量和延时变量 2.3 变量导出 2.4 Makefile 中使用 shell 命令 2.5 伪目标 2.6 产生依赖文件   3 常用函数 3.1 字符串相关 3.1.1 subst 3.1.2 patsubst 3.1.3 strip 3.1.4 f"
tags:
  - "clippings"
---
## 1 Makefile规则

```c
目标(target)…: 依赖(prerequiries)…
<tab>命令(command)
```

如果 `“依赖文件”` 比 `“目标文件”` 更加新，那么执行 `“命令”` 来重新生成 `“目标文件”` 。

命令被执行的 2 个条件：依赖文件比目标文件新，或是 目标文件还没生成。

## 2 一步一步完善 Makefile

第 1 个 Makefile，简单粗暴，效率低：

```makefile
test : main.c sub.c sub.h
　　gcc -o test main.c sub.c
```

第 2 个 Makefile，效率高，相似规则太多太啰嗦，不支持检测头文件:

```makefile
test : main.o sub.o
　　gcc -o test main.o sub.o
main.o : main.c
　　gcc -c -o main.o main.c
sub.o : sub.c
　　gcc -c -o sub.o sub.c
clean:
　　rm *.o test -f
```

第 3 个 Makefile，效率高，精炼，不支持检测头文件：

```makefile
test : main.o sub.o
　　gcc -o test main.o sub.o
%.o : %.c
　　gcc -c -o $@ $<
clean:
　　rm *.o test -f
```

第 4 个 Makefile，效率高，精炼，支持检测头文件(但是需要手工添加头文件规则)：

```makefile
test : main.o sub.o
　　gcc -o test main.o sub.o
%.o : %.c
　　gcc -c -o $@ $<
sub.o : sub.h
clean:
　　rm *.o test -f
```

## 2.1 模式规则

```makefile
%.o:%.c
　　$(CC) -c  $< -o $@
```

前面第3第4个Makefile都用到了模式规则。

## 2.2 自动变量

```makefile
$@--目标文件
$^--所有的依赖文件
$<--第一个依赖文件
$?--所有的比目标新的依赖文件
```

### 2.2.1 函数传参

[![img](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/1.png)](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/1.png) img

函数传参也属于自动变量。上图的make结果为：

[![img](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/2.png)](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/2.png) img

## 2.3 立即变量和延时变量

```makefile
A := $(C) //立即变量
B = $(C)  // 延时变量
C = abc  // 延时变量
#D = 100ask
D ?= weidongshan//延时变量，只有第一次定义时赋值才成功；如果曾定义过，此赋值无效
all:
        @echo A = $(A)
        @echo B = $(B)
        @echo D = $(D)
C += 123
```

[![img](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/3.png)](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/3.png) img

## 2.3 变量导出

`A makefile` 中的变量无法在 `B makefile` 识别，因此要用 `export` 导出如：

```makefile
export CC = $(CROSS_COMPILE)gcc
export BUILD_DIR=/home/book/100ask_imx6ull-sdk
```

## 2.4 Makefile 中使用 shell 命令

```makefile
PWD=$(shell pwd)
CP=$(shell cp)
```

## 2.5 伪目标

`.PHONY` 表示伪目标。表示无条件执行目标。makefile将不会判断该目标是否存在或者该目标是否需要更新。

```makefile
clean:
　　rm -f $(shell find -name "*.o")
　　rm -f $(TARGET)
.PHONY : clean
```

## 2.6 产生依赖文件

第5个Makefile。效率高，精炼，支持自动检测头文件：

```makefile
objs := main.o sub.o
test : $(objs)
　　gcc -o test $^
# 需要判断是否存在依赖文件
# .main.o.d .sub.o.d
dep_files := $(foreach f, $(objs), .$(f).d)
#dep_files := $(patsubst %,.%.d, $(objs))
dep_files := $(wildcard $(dep_files))
# 把依赖文件包含进来
ifneq ($(dep_files),)
include $(dep_files)
endif
%.o : %.c
gcc -Wp,-MD,.$@.d -c -o $@ $<
clean:
rm *.o test -f
distclean:
rm $(dep_files) *.o test -f
```

## 3 常用函数

## 3.1 字符串相关

### 3.1.1 subst

```makefile
$(subst from,to,text)
```

在文本 `text` 中使用 `to` 替换每一处 `from` 。

比如：

```makefile
$(subst ee,EE,feet on the street)
```

结果为 `fEEt on the strEEt`

### 3.1.2 patsubst

```makefile
$(patsubst pattern,replacement,text)
```

寻找 `text` 中符合格式 `pattern` 的字，用 `replacement` 替换它们。

`pattern` 和 `replacement` 中可以使用通配符。

比如：

```makefile
$(patsubst %.c,%.o,x.c.c bar.c)
```

结果为： `x.c.o bar.o`

### 3.1.3 strip

```makefile
$(strip string)
```

去掉前导和结尾空格，并将中间的多个空格压缩为单个空格。

比如：

```makefile
$(strip a      b c )
```

结果为： `foo.c bar.c baz.s`

### 3.1.4 findstring

```makefile
$(findstring find,in)
```

在字符串 `in` 中搜寻 `find` ，如果找到，则返回值是 `find` ，否则返回值为空。

比如：

```makefile
$(findstring a,a b c)
$(findstring a,b c)
```

将分别产生值 `a` 和(空字符串)

3.1.5 `filter`

```makefile
$(filter pattern...,text)
```

返回在 `text` 中由空格隔开且匹配格式 `pattern...`的字，去除不符合格式 `pattern...`的字。

比如：

```makefile
$(filter %.c %.s,foo.c bar.c baz.s ugh.h)
```

结果为 `foo.c bar.c baz.s` 。

### 3.1.6 filter-out

```makefile
$(filter-out pattern...,text)
```

返回在 `text` 中由空格隔开且不匹配格式 `pattern...`的字，去除符合格式 `pattern...`的字。它是函数 filter 的反函数。

比如：

```makefile
$(filter %.c %.s,foo.c bar.c baz.s ugh.h)
```

结果为 `ugh.h` 。

### 3.1.7 sort

```makefile
$(sort list)
```

将 `list` 中的字按字母顺序排序，并去掉重复的字。输出由单个空格隔开的字的列表。

比如：

```makefile
$(sort foo bar lose)
```

返回值是 `bar foo lose`

## 3.2 文件名相关

### 3.2.1 dir

```makefile
$(dir names...)
```

抽取 `names...`中每一个文件名的路径部分，文件名的路径部分包括从文件名的首字符到最后一个斜杠(含斜杠)之前的一切字符。

比如：

```makefile
$(dir src/foo.c hacks)
```

结果为 `src/ ./` 。

### 3.2.2 notdir

```makefile
$(notdir names...)
```

抽取 `names...`中每一个文件名中除路径部分外一切字符（真正的文件名）。

比如：

```makefile
$(notdir src/foo.c hacks)
```

结果为 `foo.c hacks` 。

### 3.2.3 suffix

```makefile
$(suffix names...)
```

抽取 `names...`中每一个文件名的后缀。

比如：

```makefile
$(suffix src/foo.c src-1.0/bar.c hacks)
```

结果为`.c .c` 。

### 3.2.4 basename

```makefile
$(basename names...)
```

抽取 `names...`中每一个文件名中除后缀外一切字符。

比如：

```makefile
$(basename src/foo.c src-1.0/bar hacks)
```

结果为 `src/foo src-1.0/bar hacks` 。

### 3.2.5 addsuffix

```makefile
$(addsuffix suffix,names...)
```

参数 ` names...`是一系列的文件名，文件名之间用空格隔开； suffix 是一个后缀名。将 suffix(后缀)的值附加在每一个独立文件名的后面，完成后将文件名串联起来，它们之间用单个空格隔开。

比如：

```makefile
$(addsuffix .c,foo bar)
```

结果为 `foo.c bar.c` 。

### 3.2.6 addprefix

```makefile
$(addprefix prefix,names...)
```

参数 ` names` 是一系列的文件名，文件名之间用空格隔开； prefix 是一个前缀名。将 preffix(前缀)的值附加在每一个独立文件名的前面，完成后将文件名串联起来，它们之间用单个空格隔开。

比如：

```makefile
$(addprefix src/,foo bar)
```

结果为 `src/foo src/bar` 。

### 3.2.7 wildcard

```makefile
$(wildcard pattern)
```

参数 `pattern` 是一个文件名格式，包含有通配符(通配符和 shell 中的用法一样)。函数 wildcard 的结果是一列和格式匹配的且真实存在的文件的名称，文件名之间用一个空格隔开。

比如若当前目录下有文件 1.c、 2.c、 1.h、 2.h，则：

```makefile
c_src := $(wildcard \*.c)
```

结果为 `1.c 2.c`

### 3.2.8 join

`$(join list1,list2)`

逐个地将list2中的元素链接到list1。

```makefile
LIST1 := foo bar
LIST2 := .c  .p
RESULT := ${join ${LIST1} , ${LIST2}}
```

结果为： `foo.c bar.p`

### 3.2.9realpath

`$(realpath names…)`

对 `names中` 的每个文件，求其绝对路径，当目标为链接时，将解析链接。

### 3.2.10 abspath

`$(abspath names…)`

对 `names中` 的每个文件，求其绝对路径。不解析链接。

### 3.2.11 file

`$(file op filename[,text])`

向文件执行文本的输入输出.

```makefile
TEXT := "hello world"
RESULT := \${file > test,${TEXT}}
```

前目录下存在test文件时， `"hello world"` 被写入到test中，当不存在test文件时，文件被创建且同时写入 `"hello world"`.

## 3.3 其他函数

### 3.3.1 foreach

```makefile
$(foreach var,list,text)
```

比如：

```makefile
dirs := a b c d
files := $(foreach dir,$(dirs),$(wildcard $(dir)/*))
```

这里 `text` 是 `$(wildcard $(dir)/*)` ，它的扩展过程如下：

第一个赋给变量 `dir` 的值是 `a` ， 扩展结果为 `$(wildcard a/*)` ；  
第二个赋给变量 `dir` 的值是 `b` ， 扩展结果为 `$(wildcard b/*)` ；  
第三个赋给变量 `dir` 的值是 `c` ， 扩展结果为 `$(wildcard c/*)` ；

如此继续扩展。

这个例子和下面的例有共同的结果：

```makefile
files := $(wildcard a/* b/* c/* d/*)
```

### 3.3.2 origin

```makefile
$(origin variable)
```

变量 `variable` 是一个查询变量的名称，不是对该变量的引用。所以，不能采用 `$` 和圆括号的格式书写该变量，当然，如果需要使用非常量的文件名，可以在文件名中使用变量引用。

函数 `origin` 的结果是一个字符串，该字符串变量是这样定义的：

[![img](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/4.png)](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/4.png) img

例如定义编译时用 `verbose` 还是 `quiet` 打印， `verbose` 表示输出详细过程， `quiet` 输出简略信息。

[![img](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/5.png)](https://fuzidage.github.io/2024/06/24/Makefile-%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%92%8C%E9%80%9A%E7%94%A8%E6%A8%A1%E6%9D%BF/5.png) img

将所有的信息都输出到同一个文件中：

```makefile
make xxx > build_output_all.txt 2>&1
```

### 3.3.3 word

`$(word n,text)`

返回text列表中第n个元素.

```makefile
TEXT := foo.c foo.h bar.c
RESULT := ${word 2,${TEXT}}
```

结果： `foo.h`

### 3.3.4wordlist

`$(wordlist s,e,text)`

返回text列表中指定的由s(start)开始由e(end)结尾的元素集合.

### 3.3.5 words

`$(words text)`

返回text列表中的元素数量。

### 3.3.6 firstword/lastword

`$(firstword names…) \$(lastword names…)`

返回names列表中的第一个、最后一个元素.

### 3.3.7 call

```makefile
func = $1.$2
foo = $(call func,a,b)
all:
    @echo $(foo)
```

结果： `a.b`, 子函数调用，数的参数会被赋值给临时参数·= `\$1,\$2,\$0` 则代表函数名本身.

### 3.3.8 info

```makefile
ifeq ($(KERNEL_DIR), )
$(info Please set KERNEL_DIR global variable!!)
endif

ifeq ($(INSTALL_DIR), )
INSTALL_DIR = ko
endif
CUR_DIR = $(PWD)

$(info ** [ KERNEL_DIR ] ** = $(KERNEL_DIR))
$(info ** [ INSTALL_DIR ] ** = $(INSTALL_DIR))
```

可以执行目标前先打印信息。

## 3.4 VPATH选项

VPATH中添加的目录，即使是文件处于其他目录，我们也可以像操作当前目录一样操作其他目录的文件，例如：

```makefile
VPATH += src
all:foo.c
cc $^ -o $@
```

等效于：

```makefile
all:src/foo.c
cc $^ -o $@
```

但是写成下面这样是不行的：

```makefile
VPATH += src
all:
cc foo.c -o $@
```

这是因为： `VPATH` 是 `makefile中` 的语法规则，而命令部分是由 `shell` 解析，所以 `shell` 并不会解析 `VPATH` 。

## 3.5 make环境变量

```
AR 打包程序，默认值为ar，对目标文件进行打包，封装静态库
AS 汇编程序，默认值为as，将汇编指令编译成机器指令
CC c编译器，默认值为cc，通常情况下，cc是一个指向gcc的链接，负责将c程序编译成汇编程序。
CXX c++编译器，默认值为g++
CPP 预处理器，默认值为 "$(CC) -E"，注意这里的CPP不是C++，而是预处理器。
RM 删除文件，默认值为 "rm -f"，-f表示强制删除
```

## 3.6 make编译选项

```
ARFLAGS 指定$(AR)运行时的参数，默认值为"ar"
ASFLAGS 指定$(AS)运行时的参数，无默认值
CFLAGS 指定$(CC)运行时的参数，无默认值
CXXFLAGS 指定$(CXX)运行时的参数，无默认值
CPPFLAGS 指定$(CPP)运行时的参数，无默认值，注意这里的CPP不是C++，而是预处理器
LDFLAGS 指定ld链接器运行时的参数，无默认值
LDLIBS 指定ld链接器运行时的链接库参数，无默认值。

make --debug
    输出每一步输出的详细流程，对于调试时非常方便。
```

## 4 通用型makefile

本 `makefile` 是参考linux内核的 `makefile` 框架改编简化，大家可以参考Linux内核中 `built-in.o` 的产生过程来进一步了解该流程。

顶层目录下，存在 `Makefile` 和 `Makefile.build` 两个文件。这两个文件非常重要，make命令能递归查找每个子目录，就是这2个 `Makefile` 文件的功劳。

## 4.1 顶层Makefile

```makefile
# 延时变量, 只有第一次定义赋值才成功.而该变量在/etc/profile中. 已定义为arm-linux-gnueabihf-
CROSS_COMPILE ?=
# 定义延时变量
AS                = $(CROSS_COMPILE)as
LD                = $(CROSS_COMPILE)ld
CC                = $(CROSS_COMPILE)gcc
CPP                = $(CC) -E
AR                = $(CROSS_COMPILE)ar
NM                = $(CROSS_COMPILE)nm
STRIP                = $(CROSS_COMPILE)strip
OBJCOPY                = $(CROSS_COMPILE)objcopy
OBJDUMP                = $(CROSS_COMPILE)objdump
# export全局变量, 可供其他Makefile使用
export AS LD CC CPP AR NM
export STRIP OBJCOPY OBJDUMP
CFLAGS := -Wall -O2 -g
CFLAGS += -I $(shell pwd)/include
LDFLAGS := -lpthread -lfreetype -lm
export CFLAGS LDFLAGS

TOPDIR := $(shell pwd)
export TOPDIR

TARGET := test

obj-y += display/
# obj-y += unittest/
#obj-y += input/
#obj-y += font/
obj-y += ui/
obj-y += page/
obj-y += config/
obj-y += business/

all : start_recursive_build $(TARGET)
        @echo $(TARGET) has been built!

start_recursive_build:
        @echo $@
        @echo obj-y = $(obj-y)
        make -C ./ -f $(TOPDIR)/Makefile.build

$(TARGET) : built-in.o
        $(CC) -o $(TARGET) built-in.o $(LDFLAGS)
clean:
        rm -f $(shell find -name "*.o")
        rm -f $(TARGET)
distclean:
        rm -f $(shell find -name "*.o")
        rm -f $(shell find -name "*.d")
        rm -f $(TARGET)
```

顶层 `Makefile` 的作用：

1. 提供项目make命令的执行入口，提供所有编译target目标。
2. 定义全局变量，编译选项，链接选项等。
3. 通过 `obj-y` 指定要搜索的子目录。
4. 切换目录，递归执行make命令，并执行根目录的 `Makefile.build` 文件

## 4.2 Makefile.build

```makefile
PHONY := __build echo_obj
__build:
obj-y :=
subdir-y :=
EXTRA_CFLAGS :=

#包含当前执行目录的Makefile
include Makefile

# obj-y := a.o b.o c/ d/
# $(filter %/, $(obj-y))   : c/ d/
# __subdir-y  : c d
# subdir-y    : c d
# $(filter %/, $(obj-y)) 从变量obj-y中过滤出以"/"结尾的目录名
# $(patsubst %/,%,$(filter %/, $(obj-y))) 去掉obj-y中以"/"结尾的目录名中的"/"
__subdir-y        := $(patsubst %/,%,$(filter %/, $(obj-y)))
subdir-y        += $(__subdir-y)
 
# c/built-in.o d/built-in.o
# foreach(var,list,text), 意为foreach var in list, change it to text
# 将子目录列表subdir-y中, 每一项(每个文件名)f, 都修改为f/built-in.o
# 也就是说, 每个子目录, 都会对应生成一个名为 "子目录名/built-in.o"的文件 (.o文件是链接文件)
subdir_objs := $(foreach f,$(subdir-y),$(f)/built-in.o)

# a.o b.o
# 从obj-y中过滤掉目录名(名称以"/"结尾), 只剩下普通文件(.o文件)
cur_objs := $(filter-out %/, $(obj-y))
dep_files := $(foreach f,$(cur_objs),.$(f).d)

# 如果依赖文件存在, 就列出来重新赋值给dep_files
dep_files := $(wildcard $(dep_files))

# 如果依赖文件列表不为空, 就直接包含(include)依赖文件列表
ifneq ($(dep_files),)
  include $(dep_files)
endif

# 每个子目录名(不含"/")追加到伪目标
PHONY += $(subdir-y)

echo_obj:
        @echo "********************  echo_obj  *************************"
        @echo obj-y:$(obj-y)
        @echo __subdir-y:$(__subdir-y)
        @echo subdir-y:$(subdir-y)
        @echo subdir_objs:$(subdir_objs)
        @echo cur_objs:$(cur_objs)
        @echo "**********************************************************"

__build: echo_obj $(subdir-y) built-in.o

#用顶层Makefile.build的规则去子目录去编译
$(subdir-y):
        @echo subdir-y = $@
        make -C $@ -f $(TOPDIR)/Makefile.build

# 定义built-in.o依赖规则，当subdir-y为空了才开始执行
# cur_objs 从obj-y过滤出的普通文件(.o文件)
# subdir_objs 子目录下的built-in.o
built-in.o : $(cur_objs) $(subdir_objs)
        $(LD) -r -o $@ $^

#延时变量, generate dep_file
dep_file = .$@.d

%.o : %.c
        $(CC) $(CFLAGS) $(EXTRA_CFLAGS) $(CFLAGS_$@) -Wp,-MD,$(dep_file) -c -o $@ $<
.PHONY : $(PHONY)
```

`Makefile.build` 作用：

1. 包含引用顶层Makefile。
2. 取出每个子Makefile中定义的`.o` 文件，再根据 `%.o:%.c` 模式规则，自动寻找`.c` 源码文件。
3. 取出每个子Makefile中定义的子目录，再用 `make -C` 切换到子目录，从而实现递归目录编译。
4. 为每个`.o` 文件生成依赖文件 `(.d)`,并包含进Makefile.build。
5. 为每个子目录(含有Makefile)生成一个 `built-in.o` 文件，便于根目录下的Makefile文件编译、链接。
6. 设置伪目标。

## 4.3 Make过程举例

项目目录展开如下：

```
.
├── bin
│   └── led.sh
├── business
│   ├── main.c
│   └── Makefile
├── config
│   ├── config.c
│   └── Makefile
├── display
│   ├── disp_manager.c
│   ├── framebuffer.c
│   └── Makefile
├── font
│   ├── font_manager.c
│   ├── freetype.c
│   └── Makefile
├── include
│   ├── common.h
│   ├── config.h
│   ├── disp_manager.h
│   ├── font_manager.h
│   ├── input_manager.h
│   ├── page_manager.h
│   ├── tslib.h
│   └── ui.h
├── input
│   ├── input_manager.c
│   ├── Makefile
│   ├── netinput.c
│   └── touchscreen.c
├── Makefile
├── Makefile.build
├── page
│   ├── main_page.c
│   ├── Makefile
│   └── page_manager.c
├── ui
│   ├── button.c
│   └── Makefile
└── unittest
    ├── client.c
    ├── disp_test.c
    ├── font_test.c
    ├── input_manager_test.c
    ├── Makefile
    ├── page_test.c
    └── ui_test.c
```

### 4.3.1 子目录MakeFIle-display为例

```makefile
EXTRA_CFLAGS :=
CFLAGS_file.o :=
obj-y += disp_manager.o
obj-y += framebuffer.o
```

### 4.3.2 编译详细输出日志

顶层目录输入 `make all V=1` 来看详细的编译过程：

点击查看代码
```
start_recursive_build
obj-y = display/ ui/ page/ config/ business/
make -C ./ -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[1]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command'
********************  echo_obj  *************************
obj-y: display/ ui/ page/ config/ business/
__subdir-y:display ui page config business
subdir-y: display ui page config business
subdir_objs:display/built-in.o ui/built-in.o page/built-in.o config/built-in.o business/built-in.o
cur_objs:
**********************************************************
subdir-y = display
make -C display -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/display'
********************  echo_obj  *************************
obj-y: disp_manager.o framebuffer.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:disp_manager.o framebuffer.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.disp_manager.o.d -c -o disp_manager.o disp_manager.c
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.framebuffer.o.d -c -o framebuffer.o framebuffer.c
ld -r -o built-in.o disp_manager.o framebuffer.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/display'
subdir-y = ui
make -C ui -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/ui'
********************  echo_obj  *************************
obj-y: button.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:button.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.button.o.d -c -o button.o button.c
ld -r -o built-in.o button.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/ui'
subdir-y = page
make -C page -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/page'
********************  echo_obj  *************************
obj-y: main_page.o page_manager.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:main_page.o page_manager.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.main_page.o.d -c -o main_page.o main_page.c
main_page.c:198:16: warning: ‘GetButtonByInputEvent’ defined but not used [-Wunused-function]
  198 | static Button* GetButtonByInputEvent(InputEvent *pInputEvent)
      |                ^~~~~~~~~~~~~~~~~~~~~
main_page.c:103:13: warning: ‘GenerateButtons’ defined but not used [-Wunused-function]
  103 | static void GenerateButtons(void)
      |             ^~~~~~~~~~~~~~~
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.page_manager.o.d -c -o page_manager.o page_manager.c
ld -r -o built-in.o main_page.o page_manager.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/page'
subdir-y = config
make -C config -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/config'
********************  echo_obj  *************************
obj-y: config.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:config.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.config.o.d -c -o config.o config.c
ld -r -o built-in.o config.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/config'
subdir-y = business
make -C business -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/business'
********************  echo_obj  *************************
obj-y: main.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:main.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.main.o.d -c -o main.o main.c
main.c: In function ‘main’:
main.c:12:9: warning: unused variable ‘err’ [-Wunused-variable]
   12 |     int err;
      |         ^~~
ld -r -o built-in.o main.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/business'
ld -r -o built-in.o display/built-in.o ui/built-in.o page/built-in.o config/built-in.o business/built-in.o
make[1]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command'
gcc -o test built-in.o -lpthread -lfreetype -lm
test has been built!
```

### 4.3.3 编译入口

首先编译目标 `start_recursive_build` ， 列出目标、要进行编译的子目录模块，进入当前Makefile所在目录，按照顶层 `Makefile.build` 的规则编译，打印如下：

```makefile
start_recursive_build
obj-y = display/ ui/ page/ config/ business/
```

### 4.3.4 编译\_\_build

执行顶层 `Makefile.build` ，执行目标 `__build` ，执行 `echo_obj` ， 打印:

```makefile
make -C ./ -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/
31_improve_command/Makefile.build
make[1]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command'
********************  echo_obj  *************************
obj-y: display/ ui/ page/ config/ business/
__subdir-y:display ui page config business
subdir-y: display ui page config business
subdir_objs:display/built-in.o ui/built-in.o page/built-in.o config/built-in.o business/built-in.o
cur_objs:
**********************************************************
```

#### 4.3.4.1 编译$(subdir-y)

在每个子目录产生 `build-in.o` ：

执行 `$(subdir-y)` ， 又 `$(subdir-y)=display ui page config business` ，因此目标编译规则展开如下：

```makefile
display ui page config business:
        @echo subdir-y = $@
        make -C $@ -f $(TOPDIR)/Makefile.build
```

打印如下:

```makefile
subdir-y = display
make -C display -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/display'
********************  echo_obj  *************************
obj-y: disp_manager.o framebuffer.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:disp_manager.o framebuffer.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.disp_manager.o.d -c -o disp_manager.o disp_manager.c
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.framebuffer.o.d -c -o framebuffer.o framebuffer.c
ld -r -o built-in.o disp_manager.o framebuffer.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/display'
```

分析：目标是 `display` ，进入 `display` 目录，按照顶层 `Makefile.build` 的规则编译，再次递归调用 `__build` ，可以看到只有当子目录 `subdir-y` 为空了，才不会递归进去，那么此时会执行 `built-in.o` 目标。

```makefile
built-in.o : $(cur_objs) $(subdir_objs)
        $(LD) -r -o $@ $^
```

此时按照推导规则进行 `disp_manager.o， framebuffer.o` 的编译，在 `dispaly` 目录下打包成 `build-in.o` 。

同理， `ui` 目录编译：

```makefile
subdir-y = ui
make -C ui -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/ui'
********************  echo_obj  *************************
obj-y: button.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:button.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.button.o.d -c -o button.o button.c
ld -r -o built-in.o button.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/ui'
```

同理， `page` 目录编译：

```makefile
subdir-y = page
make -C page -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/page'
********************  echo_obj  *************************
obj-y: main_page.o page_manager.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:main_page.o page_manager.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.main_page.o.d -c -o main_page.o main_page.c
main_page.c:198:16: warning: ‘GetButtonByInputEvent’ defined but not used [-Wunused-function]
  198 | static Button* GetButtonByInputEvent(InputEvent *pInputEvent)
      |                ^~~~~~~~~~~~~~~~~~~~~
main_page.c:103:13: warning: ‘GenerateButtons’ defined but not used [-Wunused-function]
  103 | static void GenerateButtons(void)
      |             ^~~~~~~~~~~~~~~
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.page_manager.o.d -c -o page_manager.o page_manager.c
ld -r -o built-in.o main_page.o page_manager.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/page'
```

同理， `config` 目录编译：

```makefile
subdir-y = config
make -C config -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/config'
********************  echo_obj  *************************
obj-y: config.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:config.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.config.o.d -c -o config.o config.c
ld -r -o built-in.o config.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/config'
```

同理， `business` 目录编译：

```makefile
subdir-y = business
make -C business -f /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/Makefile.build
make[2]: Entering directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/business'
********************  echo_obj  *************************
obj-y: main.o
__subdir-y:
subdir-y:
subdir_objs:
cur_objs:main.o
**********************************************************
gcc -Wall -O2 -g -I /media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/include   -Wp,-MD,.main.o.d -c -o main.o main.c
main.c: In function ‘main’:
main.c:12:9: warning: unused variable ‘err’ [-Wunused-variable]
   12 |     int err;
      |         ^~~
ld -r -o built-in.o main.o
make[2]: Leaving directory '/media/cvitek/robin.lee/my_test/study/weidongshan/imx6study/project/electronic_test_tools/31_improve_command/business'
```

#### 4.3.4.2 打包成总的built-in.o

最后子目录的 `built-in.o` 都生成了，再来返回顶层 `Makefile` ，此时 `subdir_objs和cur_objs` 如下：

```makefile
subdir_objs:display/built-in.o ui/built-in.o page/built-in.o config/built-in.o business/built-in.o
cur_objs:
```

那么继续：

```makefile
built-in.o : $(cur_objs) $(subdir_objs)
        $(LD) -r -o $@ $^
```
```
ld -r -o built-in.o display/built-in.o ui/built-in.o page/built-in.o config/built-in.o business/built-in.o
```

至此， `__build` 目标执行完毕。

### 4.3.5 编译出口

回到顶层 `makefile` 的 `start_recursive_build` ，回到 `all` 目标，执行 `$(TARGET)` 目标。

```makefile
gcc -o test built-in.o -lpthread -lfreetype -lm
```

最终回到 `all` 目标,打印：

```
test has been built!
```

## 5 通用型makefile2-裸机版

## 5.1 文件工程目录

```makefile
book@100ask:~/ftp/openedv/bak_drv_prj/05_ledc_bsp$ tree
├── bsp
│   ├── clk
│   │   ├── bsp_clk.c
│   │   └── bsp_clk.h
│   ├── delay
│   │   ├── bsp_delay.c
│   │   └── bsp_delay.h
│   └── led
│       ├── bsp_led.c
│       └── bsp_led.h
├── imx6ul
│   ├── cc.h
│   ├── fsl_common.h
│   ├── fsl_iomuxc.h
│   ├── imx6ul.h
│   └── MCIMX6Y2.h
├── imx6ul.lds
├── imxdownload
├── Makefile
├── obj
└── project
    ├── main.c
    └── start.S
```

## 5.2 lds链接脚本

```c
SECTIONS{
    . = 0X87800000;
    .text :
    {
        obj/start.o 
        *(.text)
    }
    .rodata ALIGN(4) : {*(.rodata*)}     
    .data ALIGN(4)   : { *(.data) }    
    __bss_start = .;    
    .bss ALIGN(4)  : { *(.bss)  *(COMMON) }    
    __bss_end = .;
}
```

## 5.3 Makefile

```makefile
CROSS_COMPILE ?= arm-linux-gnueabihf-
TARGET ?= bsp
CC := $(CROSS_COMPILE)gcc
LD := $(CROSS_COMPILE)ld
OBJCOPY := $(CROSS_COMPILE)objcopy
OBJDUMP := $(CROSS_COMPILE)objdump

INCDIRS := imx6ul \
　　　　　　bsp/clk \
　　　　　　bsp/led \
　　　　　　bsp/delay

SRCDIRS := project \
　　　　　　bsp/clk \
　　　　　　bsp/led \
　　　　　　bsp/delay

INCLUDE := $(patsubst %, -I %, $(INCDIRS))

SFILES := $(foreach dir, $(SRCDIRS), $(wildcard $(dir)/*.S))
CFILES := $(foreach dir, $(SRCDIRS), $(wildcard $(dir)/*.c))

SFILENDIR := $(notdir $(SFILES))
CFILENDIR := $(notdir $(CFILES))

SOBJS := $(patsubst %, obj/%, $(SFILENDIR:.S=.o))
COBJS := $(patsubst %, obj/%, $(CFILENDIR:.c=.o))
OBJS := $(SOBJS) $(COBJS)

VPATH := $(SRCDIRS)

.PHONY: clean

$(TARGET).bin : $(OBJS)
　　$(LD) -Timx6ul.lds -o $(TARGET).elf $^
　　$(OBJCOPY) -O binary -S $(TARGET).elf $@
　　$(OBJDUMP) -D -m arm $(TARGET).elf > $(TARGET).dis

$(SOBJS) : obj/%.o : %.S
　　$(CC) -Wall -nostdlib -c -O2 $(INCLUDE) -o $@ $<

$(COBJS) : obj/%.o : %.c
　　$(CC) -Wall -nostdlib -c -O2 $(INCLUDE) -o $@ $<

print:
　　@echo INCDIRS=$(INCDIRS)
　　@echo SRCDIRS=$(SRCDIRS)
　　@echo INCLUDE=$(INCLUDE)
　　@echo SFILES=$(SFILES)
　　@echo CFILES=$(CFILES)
　　@echo SFILENDIR=$(SFILENDIR)
　　@echo CFILENDIR=$(CFILENDIR)
　　@echo SOBJS=$(SOBJS)
　　@echo COBJS=$(COBJS)

clean:
　　rm -rf $(TARGET).elf $(TARGET).dis $(TARGET).bin $(COBJS) $(SOBJS)
```

打印出目标和依赖文件:

```makefile
book@100ask:~/ftp/openedv/bak_drv_prj/05_ledc_bsp$ make print
INCDIRS=imx6ul bsp/clk bsp/led bsp/delay
SRCDIRS=project bsp/clk bsp/led bsp/delay
INCLUDE= -I imx6ul -I bsp/clk -I bsp/led -I bsp/delay
SFILES= project/start.S
CFILES= project/main.c bsp/clk/bsp_clk.c bsp/led/bsp_led.c bsp/delay/bsp_delay.c
SFILENDIR=start.S
CFILENDIR=main.c bsp_clk.c bsp_led.c bsp_delay.c
SOBJS= obj/start.o
COBJS= obj/main.o obj/bsp_clk.o obj/bsp_led.o obj/bsp_delay.o
```

编译结果如下:

```makefile
book@100ask:~/ftp/openedv/bak_drv_prj/05_ledc_bsp$ make
arm-buildroot-linux-gnueabihf-gcc -Wall -nostdlib -c -O2   -I imx6ul  -I bsp/clk  -I bsp/led  -I bsp/delay -o obj/start.o project/start.S
arm-buildroot-linux-gnueabihf-gcc -Wall -nostdlib -c -O2   -I imx6ul  -I bsp/clk  -I bsp/led  -I bsp/delay -o obj/main.o project/main.c
arm-buildroot-linux-gnueabihf-gcc -Wall -nostdlib -c -O2   -I imx6ul  -I bsp/clk  -I bsp/led  -I bsp/delay -o obj/bsp_clk.o bsp/clk/bsp_clk.c
arm-buildroot-linux-gnueabihf-gcc -Wall -nostdlib -c -O2   -I imx6ul  -I bsp/clk  -I bsp/led  -I bsp/delay -o obj/bsp_led.o bsp/led/bsp_led.c
arm-buildroot-linux-gnueabihf-gcc -Wall -nostdlib -c -O2   -I imx6ul  -I bsp/clk  -I bsp/led  -I bsp/delay -o obj/bsp_delay.o bsp/delay/bsp_delay.c
arm-buildroot-linux-gnueabihf-ld -Timx6ul.lds -o bsp.elf obj/start.o obj/main.o obj/bsp_clk.o obj/bsp_led.o obj/bsp_delay.o
arm-buildroot-linux-gnueabihf-objcopy -O binary -S bsp.elf bsp.bin
arm-buildroot-linux-gnueabihf-objdump -D -m arm bsp.elf > bsp.dis
```