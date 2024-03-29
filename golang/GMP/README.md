# 						GMP学习笔记

## 老调度器的缺点：

1、创建、销毁、调度 G 都需要每个 M 获取锁，这就形成了激烈的锁竞争。
2、M 转移 G 会造成延迟和额外的系统负载。比如当 G 中包含创建新协程的时候，M 创建了 G’，为了继续执行 G，需要把 G’交给 M’执行，也造成了很差的局部性，因为 G’和 G 是相关的，最好放在 M 上执行，而不是其他 M’。

3、系统调用 (CPU 在 M 之间的切换) 导致频繁的线程阻塞和取消阻塞操作增加了系统开销。

### GMP模型简介

面对之前调度器的问题，Go 设计了新的调度器，即GMP模型。

G - goroutine 协程

P - processor 处理器

M - thread 内核线程

在新调度器中，除了 M (thread) 和 G (goroutine)，又引进了 P (Processor)。

**Processor，它包含了运行 goroutine 的资源**，如果线程想运行 goroutine，必须先获取 P，P 中还包含了可运行的 G 队列。

![16-GMP-调度.png](https://cdn.learnku.com/uploads/images/202003/11/58489/Ugu3C2WSpM.jpeg!large)

1、全局队列（Global Queue）：存放等待运行的 G。

2、P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G时，**G优先加入到 P 的本地队列**，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。

3、P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。

4、M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。






### 调度器设计策略

#### 1.复用线程

1）work stealing 机制

 当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。

2）hand off 机制

 当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

#### 2.利用并行

GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度，比如 GOMAXPROCS = 核数/2，则最多利用了一半的 CPU 核进行并行。

#### 3.抢占

在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方。

#### 4.全局G队列

在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。