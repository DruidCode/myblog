---
title: 一个简单的makefile编写教程
date: 2016-07-15 10:39:00
tags: 翻译
categories: c
---

最近在学习编写一个简单的web server，用到了makefile。
<!--more-->

翻译资料：
>http://www.cs.colby.edu/maxwell/courses/tutorials/maketutor/

官方文档：
>http://www.gnu.org/software/make/manual/make.html#Makefile-Contents

Makefiles是一种简单的组织代码编写方式。这个教程不仅仅是对make用法的一个初浅的探讨，也可以作为初学者的指南，可以给小到中型项目创建自己的makefiles文件。

比如有以下三个文件：

#hellomake.c

```c
#include <hellomake.h>

int main() {
  // call a function in another file
  myPrintHelloMake();

  return(0);
}
```

#hellofunc.c

```c
#include <stdio.h>
#include <hellomake.h>

void myPrintHelloMake(void) {

  printf("Hello makefiles!\n");

  return;
}
```

#helomake.h

```c
/*
example include file
*/

void myPrintHelloMake(void);
```

通常，你将执行下面的命令来编译这些代码：

    gcc -o hellomake hellomake.c hellofunc.c -I.
    
这里编译了两个c文件，命名了可执行文件hellomake。-I.的意思是包含在当前目录(.)下的hellomake.h文件。典型的测试／修改／debug的方法是在终端用向上箭头来回到你最近使用的编译命令，所以你不得不每次都这样用，特别是一旦你增加了一些c文件。

不幸的事，这种编译方法有两个弊端。第一，如果你丢了编译命令或者换了电脑，你不得不从零输入，效率极低。第二，如果你只是修改了一个c文件，每次都要把它们重新编译也是耗时间和效率低的。所以，是时候看看我们用一个makefile来解决这些问题了。

你可以创建一个最简单的makefile，如下：
```c
hellomake: hellomake.c hellofunc.c
     gcc -o hellomake hellomake.c hellofunc.c -I.
```
如果你把上面命令输入到一个文件叫Makefile活着makefile，然后在命令行输入make，它将会执行你在makefile文件里写的编译命令。注意到make没有参数。而且，把文件列表放到：后面，make知道如果任何一个文件改变了，规则hellomake都需要被执行。此刻，你已经解决了问题一，避免重复用键盘上的向上箭头来寻找你最近使用的编译命令。但是，仅仅改变最近编译项，系统仍然效率不高。

一个非常重要需要注意的是在gcc命令前面有一个tab，这是在任何命令开始前都是必需的，否则make会很不开心。

为了效率更高一点，让我们试试下面的makefile：
```c
CC=gcc
CFLAGS=-I.

hellomake: hellomake.o hellofunc.o
     $(CC) -o hellomake hellomake.o hellofunc.o -I.
```
所以现在我们已经定义了一些常量CC和CFLAGS。这些都是特殊的常量，告诉make我们想如何编译文件hellomake.c和hellofunc.c。尤其是，宏指令CC是是c编译程序，而CFLAGS是传给编译命令的标记列表。把目标文件－hellomake.o和hellofunc.o－放入依赖列表和规则中，make知道首先应该编译.c版本文件，然后建立可执行文件hellomake。

对于大部分小型项目来说用这种makefile足够了。但是，还有一件事漏掉了：头文件的包含。如果你打算修改hellomake.h文件，例如，make将不会重新编译.c文件，虽然它们是必需的。为了解决这个，我们需要告诉make所有的.c文件都依赖某个.h文件。我们可以通过写一个简单的规则，加到makefile里面来实现它。

```c
CC=gcc
CFLAGS=-I.
DEPS = hellomake.h

%.o: %.c $(DEPS)
	$(CC) -c -o $@ $< $(CFLAGS)

hellomake: hellomake.o hellofunc.o 
	gcc -o hellomake hellomake.o hellofunc.o -I.
```
第一个新添加的是宏命令DEPS，设置.c文件依赖的.h文件。然后我们定义了一个规则应用于所有以.o结尾的文件。规则是：.o文件依赖.c文件，.h文件包含在DEPS宏命令中；创建.o文件，make必须用CC宏命令定义的编译器编译.c文件。-c标记是建立目标文件，-o $@表示把编译的输出放到:左边名字的文件中，而$<是依赖列表中的第一项，而宏命令CFLAGS正如上面所说的。

最终的简化，用特别宏命令$@和$^，分别表示:左边和右边，分别使以上编译规则更加通用。在下面的例子中，所有包含文件应当在DEPS列出，所有目标文件应当在OBJ中列出。
```c
CC=gcc
CFLAGS=-I.
DEPS = hellomake.h
OBJ = hellomake.o hellofunc.o 

%.o: %.c $(DEPS)
	$(CC) -c -o $@ $< $(CFLAGS)

hellomake: $(OBJ)
	gcc -o $@ $^ $(CFLAGS)
```
所以要是我们想把.h文件放入include目录，源代码放入src目录，一些本地库文件放入lib目录会怎么样呢？再者，有时候我们能隐藏那些烦人的到处都是的.o文件吗？答案是，当然可以！随后的makefile定义了include和lib目录的路径，并且把object文件放到src目录下的obj子目录。也有一个宏命令定义了任何你想要包含的库文件，比如math库文件-lm。这个makefile应该放在src目录本地。注意到，也包含了一个清理源代码和object目录的规则make clean。.PHONY规则使make命令不会生成一个名字叫clean的文件。
```c
IDIR =../include
CC=gcc
CFLAGS=-I$(IDIR)

ODIR=obj
LDIR =../lib

LIBS=-lm

_DEPS = hellomake.h
DEPS = $(patsubst %,$(IDIR)/%,$(_DEPS))

_OBJ = hellomake.o hellofunc.o 
OBJ = $(patsubst %,$(ODIR)/%,$(_OBJ))


$(ODIR)/%.o: %.c $(DEPS)
	$(CC) -c -o $@ $< $(CFLAGS)

hellomake: $(OBJ)
	gcc -o $@ $^ $(CFLAGS) $(LIBS)

.PHONY: clean

clean:
	rm -f $(ODIR)/*.o *~ core $(INCDIR)/*~ 
```
现在你有了一个完美的makefile，可以修改管理小到中型软件工程。你可以增加多个规则到makefile；甚至创建规则调用规则。想获取更多关于makefiles和make方法的信息，可以到官网[GNU Make Manual](http://www.gnu.org/software/make/manual/make.html)查看。