---
title: "* 使用Trace Profiling分析Java锁竞争"
description: "使用Trace Profiling分析Java锁竞争"
lead: ""
date: 2022-11-10T10:16:39+08:00
lastmod: 2022-11-10T10:16:39+08:00
draft: true
images: []
type: docs
menu:
  docs:
    parent: "use-cases"
    identifier: "observe-java-lock-d0ab4f618705ed7b1279e1639840fa1b"
weight: 010
toc: true
---

> 📌 要启用该功能，请先[安装Kindling](/docs/installation/kindling-agent/requirements/)，再[启用Trace Profiling功能](/docs/usage/enable-trace-profiling/)。
> 

在编程时，我们常常使用锁来保护临界资源，但使用锁意味着当有多个线程想要获取到锁时，线程间就会发生竞争，导致线程进入阻塞状态。如果开发者不恰当地使用了锁，可能会导致程序性能下降，无法充分利用CPU资源。

Kindling的Trace Profiling功能能够让用户直观地观察到Java程序的锁竞争情况，帮助用户定位问题代码、高效地解决问题。本篇文章将展示如何使用Kindling来观察Java程序的锁竞争情况，案例中的程序基于Spring Boot完成，使用Undertow作为容器，程序作为服务端对外提供API。

## 案例展示
### 启动
首先通过访问`http://${IP}:9504`打开Trace Profiling功能的主页面，点击右上角的`启动Trace检测`来开启该功能。

![](./media/202211/2022-11-01_175506_575368.gif)

启动功能后，Kindling探针会自动对其所在主机节点上的Java程序进行慢请求分析，分析的结果可以在页面上方的筛选框中找到。

![](./media/202211/2022-11-01_175519_725826.gif)

选中筛选框中的一个分析结果，可以得到下面的Trace Profiling图像。


![](./media/202211/image-trace_1667297146.png)

### 分析
其中从下图的头部信息中可以得知：

![](./media/202211/2022-11-01_175916_100969.gif)

- 该请求的协议类型为**HTTP**协议
- HTTP请求Path为`/sync-lock/1`
- Tracing的Transaction(Trace) ID为`88786150748bd45008f7510696bce47c4^1247`，可以在Tracing系统中查询相应的Trace。
- 请求的响应时间为2秒
- HTTP响应的状态码是200

从下图的耗时分解统计中可以看到，lock类型的时间为998.32ms，占用了总耗时的49.89%，这说明该请求有近乎一半的时间花费在等待Java锁上；另一半时间花费在futex上，该类型的耗时一般为非锁的阻塞等待耗时，例如程序睡眠。而该请求真正在CPU上执行的时间只有0.68ms。

![](./media/202211/2022-11-01_180100_539145.gif)

接下来，我们来检查线程在等待哪个锁以及为什么会发生等待。下图为线程列表图，注意其中`Trace分析`按钮已经高亮，这表明当前只展示了与本次请求相关的线程。默认情况下，为了减少无关线程的干扰，该选项会自动开启。

![](./media/202211/2022-11-01_180119_333857.gif)

从上图中我们可以得知：
1. XNIO-1 I/O-3 线程在该时刻读取到请求
2. XNIO-1 task-2 线程在该时刻发送回响应
3. XNIO-1 task-2 线程在黄色区间等待锁

我们可以通过点击位置3的黄色区块来查看具体的锁信息，如下图所示。

![](./media/202211/2022-11-01_180151_028378.gif)

从上图中我们可以得知：
1.该线程在等待的锁由XNIO-1 task-1线程持有。
等待获取锁的代码堆栈最上层为`com/harmonycloud/stuck/web/WaitController`包的`syncLock`方法。

我们看到，有一个线程列表中不存在的线程“XNIO-1 task-1”出现了，此时我们可以取消“Trace分析”按钮的高亮来查看此线程。如下图所示。

![](./media/202211/2022-11-01_180357_557953.gif)

点击XNIO-1 task-1的黄色区块可以看到，该线程也会执行与XNIO-1 task-2相同的代码，这两个线程在互相竞争获取锁。

## 案例分析
实际上该案例中XNIO-1 task-1线程与XNIO-1 task-2线程在处理相同Path的HTTP请求，它们执行的代码是完全相同的，它们需要访问相同的临界资源，因此使用锁来保证资源的安全。

通过Kindling的Trace Profiling功能，我们可以清晰地看到线程间在互相等待，能够快速地发现请求在等待锁上花费了一半的时间，帮助用户提高效率。

