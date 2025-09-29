---
layout: blog
title: MIT 6.S081 | Lab util
date: 2025-09-28 18:02:33
categories: 'MIT 6.S081笔记'
tags: ['刷课', '计算机', '操作系统']
permalink: /mit-6s081/lab-util/
---
## Sleep

Sleep 就是完成一个工具：`sleep [time]`，输入后休眠 `time` 秒。

完成这个工具需要掌握两个关键点：

1. 如何在 shell 中给程序传递参数
2. Sleep 的原理

### Main 函数传参

C 语言中，Main 函数要么传递参数，要么传递参数。下面的示例写的很清楚。第一种不传递参数，直接在参数列表写一个 void ，不写也是可以的；第二种传递参数看起来就比较迷惑了。

```c
int main(void) {
   // 无参数形式
   return 0;
}
int main(int argc, char *argv[]) {
   // 带参数形式
   return 0;
}
```

`int argc, char *argv[]`，这到底传递的是什么？Argc 其实是参数的个数，agrv 则是参数的指针。也就是说我们 main 函数需要一个大小为 `agrc` 的列表，名字叫 `argv`，范围是 `0-agrc-1` 。

比如说我们执行 `cp demo.c demo.cpp`，`agrc` 为 3，`argv` 为  `['cp', 'demo.c', 'demo.cpp']`。

那么 `argc` 和  `argv` 是如何得出的？这是 shell 的解析器在负责，不需要我们一个一个数参数到底有多少个。

总结一下：我们输入的指令，会被 shell 解析成 `argc` 和  `argv` ，然后交给 main 函数来使用。这里面还有更具体的流程，后面再说。

### Sleep 的原理

实验讲义里提示说让我们看看 `sysproc.c` 和 `user.h`。这就是实现 `sleep` 底层的代码。

`user.h` 提到了 sleep 系统调用的定义：`int sleep(int);`。传递一个整型，表示要休眠的时钟滴答数；返回一个整型返回值，返回 0 表示正常运行，返回 -1 表示非正常运行。这个 sleep 系统调用则是包装了下面的 `sys_sleep`。

`sysproc.c` 描述了 `sys_sleep` 实现：

```c
uint64 sys_sleep(void)
{
	int n;
	uint ticks0;
	// 参数不对，返回 -1 直接退出
	if(argint(0, &n) < 0) {
		return -1;
	}
	// 读写全局变量 ticks 前加锁，防止并发。
	acquire(&tickslock);
	// 通过读取 ticks 全局变量，ticks0 变量存储当前时间
	ticks0 = ticks;
	// 不停地检查时间，超过要求的时间就退出循环，这叫*忙等*
	while(ticks - ticks0 < n) {
		// 一旦中断，释放锁退出，返回 -1
		if(myproc()->killed) {
			release(&tickslock);
			return -1;
		}
		// sleep函数，这个函数可以让程序让出锁，避免占用资源
		sleep(&ticks, &tickslock);
	}
	// 退出循环，释放锁，返回0退出
	release(&tickslock);
	return 0;
}
```

这个系统调用有个好处：忙等检查和中间的让出机制，避免了程序持续对系统资源占用。想一想一个人在图书馆睡大觉，其他人还找不到图书馆的位置，这样是不是占用图书馆的资源呢？

### 解法

现在知道了程序如何传参，以及系统调用 sleep 是如何实现的，现在我们就动手来写自己的 `sleep` 实现。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
	int n;
	// 判断参数个数是否正确
	if(argc != 2) {
		fprintf(2, "usage: sleep <ticks>\n");
		exit(1);
	}
	// ticks 参数转换为整型
	n = atoi(argv[1]);
	// 判断 ticks 是否为正数
	if(n <= 0) {
		fprintf(2, "sleep: ticks must be positive\n");
		exit(1);
	}
	// sleep，发现返回值不对也需要报错
	if(sleep(n) != 0) {
		fprintf(2, "sleep: interrupted\n");
		exit(1);
	}
	exit(0);
}
```

网上的解答非常多，都可以多多参照。我的解法重点在于异常的处理。
