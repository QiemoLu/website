GO代码组织


引用和指针的区别？  
Map slice chan func interface 三种引用类型    
引用：某块内存的别名  
指针：指向内存的地址  
值传递 引用传递 指针传递？  
引用类型不一定是引用传递。引用传递一定是引用类型 因为有些引用类型结构体里面包含指针成员。  
Go里面只有值传递。指针也是值的一种 一个指针赋予了两个变量，值都是指针，但是存放的地址不同  

行参 实参 ？  
        形参变量只有在函数被调用时才会分配内存，调用结束后，立刻释放内存，所以形参变量只有在函数内部有效  
```
int sum(int m, int n) { //m n 形参
int i; 
for (i = m+1; i <= n; ++i) {
m += i;
}
return m;
}
```
Go 内存分配？  
Go 语言的内存分配由标准库自动完  
Go 语言的标准库函数 runtime.newobject() 用于在 heap(堆) 上的内存分配和代理 runtime.mallocgc，  
TCMalloc google开发的内存分配算法库  
核心思想就是把内存分为多级管理，从而降低锁的粒度。它将可用的堆内存采用二级分配的方式进行管理：每个线程都会自行维护一个独立的内存池，进行内存分配时优先从该内存池中分配，当内存池不足时才会向全局内存池申请，以避免不同线程对全局内存池的频繁竞争。  

内存管理组件  
内存分配由内存分配器完成。分配器由3种组件构成：mcache, mcentral, mheap。  
mcache：每个工作线程都会绑定一个mcache，本地缓存可用的mspan资源，这样就可以直接给Goroutine分配，因为不存在多个Goroutine竞争的情况，所以不会消耗锁资源。  
mcache在初始化的时候是没有任何mspan资源的，在使用过程中会动态地从mcentral申请，之后会缓存下来。当对象小于等于32KB大小时，使用mcache的相应规格的mspan进行分配。  

mcentral：为所有mcache提供切分好的mspan资源。每个central保存一种特定大小的全局mspan列表，包括已分配出去的和未分配出去的。 每个mcentral对应一种mspan，而mspan的种类导致它分割的object大小不同。当工作线程的mcache中没有合适（也就是特定大小的）的mspan时就会从mcentral获取。  

mcentral被所有的工作线程共同享有，存在多个Goroutine竞争的情况，因此会消耗锁资源。结构体定义：  

mheap：代表Go程序持有的所有堆空间，Go程序使用一个mheap的全局对象_mheap来管理堆内存。  

当mcentral没有空闲的mspan时，会向mheap申请。而mheap没有资源时，会向操作系统申请新内存。mheap主要用于大对象的内存分配，以及管理未切割的mspan，用于给mcentral切割成小对象。  

同时我们也看到，mheap中含有所有规格的mcentral，所以，当一个mcache从mcentral申请mspan时，只需要在独立的mcentral中使用锁，并不会影响申请其他规格的mspan。  

内存总结
Go在程序启动时，会向操作系统申请一大块内存，之后自行管理。  
Go内存管理的基本单元是mspan，它由若干个页组成，每种mspan可以分配特定大小的object。  
mcache, mcentral, mheap是Go内存管理的三大组件，层层递进。mcache管理线程在本地缓存的mspan；mcentral管理全局的mspan供所有线程使用；mheap管理Go的所有动态分配内存。  
极小对象会分配在一个object中，以节省资源，使用tiny分配器分配内存；一般小对象通过mspan分配内存；大对象则直接由mheap分配内存。  
内存分配栈和堆的区别？  
我记得栈的分配是编译期决定的，堆的话需要运行时去申请。  
两个的存在都是为了提高效率，  
栈，栈的内存必须是连续的，大小有限，会溢出，查找快，必须是已知大小的局部变量和函数参数。  
堆用于大内存分配。不会溢出，用于引用类型，内存非联系需要回收。  

什么是内存逃逸？  
变量太大的时候 从栈逃到堆  

指针逃逸：在方法内把局部变量指针返回   
动态类型逃逸（不确定长度大小）（在 interface 类型上调用方法）  

发送指针或带有指针的值到 channel 中  

堆是一块没有特定结构，也没有固定大小的内存区域，可以根据需要进行调整。全局变量，内存占用较大的局部变量，函数调用结束后不能立刻回收的局部变量都会存在堆里面。变量在堆上的分配和回收都比在栈上开销大的多。对于 go 这种带 GC 的语言来说，会增加 gc 压力，同时也容易造成内存碎片。  
 go语言编译器会自动决定把一个变量放在栈还是放在堆，编译器会做逃逸分析(escape analysis)，当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆。  
Gc负责回收的就是堆上的内存  

Go gc 回收？  
什么是垃圾回收？  
当程序向操作系统申请的内存不再需要时，垃圾回收主动将其回收并供其他代码进行内存申请时候复用，或者将其归还给操作系统，这种针对内存级别资源的自动回收过程，即为垃圾回收。而负责垃圾回收的程序组件，即为垃圾回收器。  

1.3以前 标记清除法go runtime在一定条件下（内存超过阈值或定期如2min）写屏障，暂停所有任务的执行，进行mark-sweep操作，操作完成后启动所有任务的执行, 从根对象就是第一个开始标记, 没有标记上的会被清除  
1.3 版本进行了一下改进，把 清除 改为了并行操作。  
先暂停所有任务执行并启动mark，mark完成后马上就重新启动被暂停的任务了，而是让sweep任务和普通协程任务一样并行的和其他任务一起执行  
1.5三色标记法(与用户代码并发执行，达到无感知)  
白色：尚未访问过。  
灰色：本对象已访问过，但是本对象 引用到 的其他对象 尚未全部访问完。全部访问后，会转换为黑色。  
黑色：本对象已访问过，而且本对象 引用到 的其他对象 也全部访问过了。
初始时，所有对象都在 【白色集合】中；  
将GC Roots 直接引用到的对象 挪到 【灰色集合】中；
从灰色集合中获取对象：  
3.1. 将本对象 引用到的 其他对象 全部挪到 【灰色集合】中；  
3.2. 将本对象 挪到 【黑色集合】里面。  
重复步骤3，直至【灰色集合】为空时结束。  
结束后，仍在【白色集合】的对象即为GC Roots 不可达，可以进行回收。  

多标和漏标  
多标：如果使用完引用对象后手动重新清零，那么变为灰色,等到下一轮gc重新遍历
漏标：断开子对象引用后重新引用：会被误回收  
读写屏障 解决漏标。  

根对象到底是什么  
根对象在垃圾回收的术语中又叫做根集合，它是垃圾回收器在标记过程时最先检查的对象，包括：  
全局变量：程序在编译期就能确定的那些存在于程序整个生命周期的变量。
执行栈：每个 goroutine 都包含自己的执行栈，这些执行栈上包含栈上的变量及指向分配的堆内存区块的指针。  
寄存器：寄存器的值可能表示一个指针，参与计算的这些指针可能指向  

常见的gc实现方式？  
追踪式，分为多种不同类型，例如：  
标记清扫：从根对象出发，将确定存活的对象进行标记，并清扫可以回收的对象。
标记整理：为了解决内存碎片问题而提出，在标记过程中，将对象尽可能整理到一块连续的内存上。  
增量式：将标记与清扫的过程分批执行，每次执行很小的部分，从而增量的推进垃圾回收，达到近似实时、几乎无停顿的目的。  
增量整理：在增量式的基础上，增加对对象的整理过程。  
分代式：将对象根据存活时间的长短进行分类，存活时间小于某个值的为年轻代，存活时间大于某个值的为老年代，永远不会参与回收的对象为永久代。并根据分代假设（如果一个对象存活时间不长则倾向于被回收，如果一个对象已经存活很长时间则倾向于存活更长时间）对对象进行回收。  
引用计数：根据对象自身的引用计数来回收，当引用计数归零时立即回收。  
GC触发的时机？  
手动触发  
被动触发，分为两种方式：  
使用系统监控，当超过两分钟没有产生任何 GC 时，强制触发 GC。  
使用步调（Pacing）算法，其核心思想是控制内存增长的比例。  

有了GC为什么还会内存泄漏？  
涉及全局变量 和常驻线程，比如无限创建线程。  
Go 的垃圾回收器有哪些相关的 API？其作用分别是什么  
runtime.GC：手动触发 GC  
runtime.ReadMemStats：读取内存相关的统计信息，其中包含部分 GC 相关的统计信息  
debug.FreeOSMemory：手动将内存归还给操作系统  
debug.ReadGCStats：读取关于 GC 的相关统计信息  
debug.SetGCPercent：设置 GOGC 调步变量  
debug.SetMaxHeap（尚未发布[10]）：设置 Go 程序堆的上限值  
chan什么时候引发异常？  
close掉继续写入，close掉读取会读到nil 不会异常  
Chan 什么情况死锁？  
无缓冲情况 先写入—再读取  


GMP 并发调度模型  
G:  goroutine  
M: machine work thread 线程  
P: processor 逻辑处理器  
每个 Goroutine （G）对应一个系统线程（M）和其所被分配的逻辑处理器核心（P）  

全局队列（Global Queue）：存放等待运行的G。  

P的本地队列：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列。  

P列表：所有的P都在程序启动时创建，并保存在数组中，最多有GOMAXPROCS(可配置)个。  

M：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列拿一批G放到P的本地队列，或从其他P的本地队列偷一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。  
Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线  程分配到CPU的核上执行。  

有关P和M的个数问题？  

1、P的数量  

由启动时环境变量$GOMAXPROCS或者是由runtime的方法GOMAXPROCS()决定。这意味着在程序执行的任意时刻都只有$GOMAXPROCS个goroutine在同时运行。  


2、M的数量  

go语言本身的限制：go程序启动时，会设置M的最大数量，默认10000.但是内核很难支持这么多的线程数，所以这个限制可以忽略。  

runtime/debug中的SetMaxThreads函数，设置M的最大数量  

一个M阻塞了，会创建新的M。  

M与P的数量没有绝对关系，一个M阻塞，P就会去创建或者切换另一个M，所以，即使P的默认数量是1，也有可能会创建很多个M出来。  

3、P和M何时会被创建  
P何时创建：在确定了P的最大数量n后，运行时系统会根据这个数量创建n个P。
M何时创建：没有足够的M来关联P并运行其中的可运行的G。比如所有的M此时都阻塞住了，而P中还有很多就绪任务，就会去寻找空闲的M，而没有空闲的，就会去创建新的M。  
 协程跟线程是有区别的，线程由CPU调度是抢占式的，协程由用户态调度是协作式的，一个协程让出CPU后，才执行下一个协程。 
即使有协程阻塞，该线程的其他协程也可以被runtime调度，转移到其他可运行的线程上。
为什么GMP要有p?  
最开始是没有p， 多线程访问同一资源需要加锁进行保证互斥/同步，所以全局G队列是有互斥锁进行保护的。  
老调度器有几个缺点：  

创建、销毁、调度G都需要每个M获取锁，这就形成了激烈的锁竞争。  
M转移G会造成延迟和额外的系统负载。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了很差的局部性，因为G’和G是相关的，最好放在M上执行，而不是其他M'。  
系统调用(CPU在M之间的切换)导致频繁的线程阻塞和取消阻塞操作增加了系统开销。
新的调度器中加入了processor  
在Go中，线程是运行goroutine的实体，调度器的功能是把可运行的goroutine分配到工作线程上。  
Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行。  

调度器的设计策略  

复用线程：避免频繁的创建、销毁线程，而是对线程的复用。  

1）work stealing机制  

 当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。  

2）hand off机制  

 当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。  

利用并行：GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在多个CPU上同时运行。GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS = 核数/2，则最多利用了一半的CPU核进行并行。  

抢占：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方。  

全局G队列：在新的调度器中依然有全局G队列，但功能已经被弱化了，当M执行work stealing从其他P偷不到G时，它可以从全局G队列获取G。  

 1、我们通过 go func()来创建一个goroutine；  

 2、有两个存储G的队列，一个是局部调度器P的本地队列、一个是全局G队列。新创建的G会先保存在P的本地队列中，如果P的本地队列已经满了就会保存在全局的队列中；  

 3、G只能运行在M中，一个M必须持有一个P，M与P是1：1的关系。M会从P的本地队列弹出一个可执行状态的G来执行，如果P的本地队列为空，就会想其他的MP组合偷取一个可执行的G来执行；  

 4、一个M调度G执行的过程是一个循环机制；  

 5、当M执行某一个G时候如果发生了syscall或则其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个P；  

 6、当M系统调用结束时候，这个G会尝试获取一个空闲的P执行，并放入到这个P的本地队列。如果获取不到P，那么这个线程M变成休眠状态， 加入到空闲线程中，然后这个G会被放入全局队列中。  

Chan 有缓存跟无缓存区别  
无缓冲的与有缓冲channel有着重大差别，那就是一个是同步的 一个是非同步的
无缓冲的chan 消费者必须准备好后 发送者才能工作 否则死锁   
有缓存的chan 消费者跟不上发送者 ，缓存占满的情况也会阻塞 死锁  
Make new 区别？  
都在 堆上分配  


New 返回一个指向该类型内存地址的指针。同时请注意它同时把分配的内存置为零，也就是类型的零值。string 就是“” 数字就是0  
Make : 只用于 chan map slice 返回这三个类型本身， 引用类型 ，把内存初始化置为nil。  

Defer的知识点  
遇到panic时，遍历本协程的defer链表，并执行defer。在执行defer过程中:遇到recover则停止panic，返回recover处继续往下执行。如果没有遇到recover，遍历完本协程的defer链表后，向stderr抛出panic信息。  
1.执行顺序 后进先出  
2.return之后的语句先执行，defer后的语句后执行. defer可以修改return的值  
3.defer遇到panic. Defer - > recover-> panic , 如果有recover则不捕获  

Chan 内部结构？  
比较重要的成员  

```
type hchan struct {
        qcount   uint           // buffer 中已放入的元素个数
        dataqsiz uint           // 用户构造 channel 时指定的 buf 大小
        buf      unsafe.Pointer // buffer
        elemsize uint16         // buffer 中每个元素的大小
        closed   uint32         // channel 是否关闭，== 0 代表未 closed
        elemtype *_type         // channel 元素的类型信息
        sendx    uint           // buffer 中已发送的索引位置 send index
        recvx    uint           // buffer 中已接收的索引位置 receive index
        recvq    waitq          // 等待接收的 goroutine  list of recv waiters
        sendq    waitq          // 等待发送的 goroutine list of send waiters

        lock mutex
}
```

buffer 相关的属性。例如 buf、dataqsiz、qcount 等。 当 channel 的缓冲区大小不为 0 时，buffer 中存放了待接收的数据。使用 ring buffer 实现。
waitq 相关的属性，可以理解为是一个 FIFO 的标准队列。其中 recvq 中是正在等待接收数据的 goroutine，sendq 中是等待发送数据的 goroutine。waitq 使用双向链表实现。  
其他属性，例如 lock、elemtype、closed 等。  

向chan发送数据 runtime.chansend 函数的实现。  
在尝试向 channel 中发送数据时，如果 recvq 队列不为空，则首先会从 recvq 中头部取出一个等待接收数据的 goroutine 出来。并将数据直接发送给该 goroutine。则调用 send 函数将数据拷贝到对应的 goroutine 的堆栈上。  
当某个 goroutine 使用 recv 操作 (例如，x := <- c)，如果此时 channel 的缓存中没有数据  
重回队列等待  
这里需要注意的是，channel 的整个发送过程和接收过程都使用 runtime.mutex 进行加锁  
解决的是并发写 并发读  
总结  

检查 recvq 是否为空，如果不为空，则从 recvq 头部取一个 goroutine，将数据发送过去，并唤醒对应的 goroutine 即可。  
如果 recvq 为空，则将数据放入到 buffer 中。  
如果 buffer 已满，则将要发送的数据和当前 goroutine 打包成 sudog 对象放入到 sendq 中。并将当前 goroutine 置为 waiting 状态。  

环形队列结构  
```
type Queen struct { 
Length int64 //队列长度 
Capacity int64 //队列容量 
Head int64 //队头 第一个元素索引
Tail int64 //队尾 D 最后一个元素索引
ata []interface{} //存放数据列表
} 
```
容量等于长度 队列已满

自旋操作
不断重试等待获取锁
自旋锁避免了操作系统进程调度和线程切换，所以自旋锁通常适用在时间比较短的情况下。
自旋锁会非常耗费性能，它阻止了其他线程的运行和调度
Go互斥锁原理
go里面没有重入锁
Mutex 的实现主要借助了 CAS(比较替换) 指令 + 自旋 + 信号量来实现
type Mutex struct {
    state int32 //状态
    sema  uint32 //信号量 控制保证锁不被多个goroutine获取
}
互斥锁有两种状态：正常状态和饥饿状态。

在正常状态下，所有等待锁的goroutine按照FIFO顺序等待。唤醒的goroutine不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。新请求锁的goroutine具有优势：它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面。 如果一个等待的goroutine超过1ms没有获取锁，那么它将会把锁转变为饥饿模式。

在饥饿模式下，锁的所有权将从unlock的gorutine直接交给交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。

如果一个等待的goroutine获取了锁，并且满足一以下其中的任何一个条件：(1)它是队列中的最后一个；(2)它等待的时候小于1ms。它会将锁的状态转换为正常状态。

正常状态有很好的性能表现，饥饿模式也是非常重要的，因为它能阻止尾部延迟的现象。


几种状态

mutexLocked — 表示互斥锁的锁定状态；
mutexWoken — 表示从正常模式被从唤醒；
mutexStarving — 当前的互斥锁进入饥饿状态；
waitersCount — 当前互斥锁上等待的 goroutine 个数 
为了保证锁的公平性，设计上互斥锁有两种状态：正常状态和饥饿状态。
同时，尝试获取锁的goroutine也有状态，有可能它是新来的goroutine，也有可能是被唤醒的goroutine, 可能是处于正常状态的goroutine, 也有可能是处于饥饿状态的goroutine。
mutex 的 state 有 32 位，它的低 3 位分别表示 3 种状态：唤醒状态、上锁状态、饥饿状态，剩下的位数则表示当前阻塞等待的 goroutine 数量。
mutex 会根据当前的 state 状态来进入正常模式、饥饿模式或者是自旋。
atomic.CompareAndSwapInt32 自旋  汇编实现
当 mutex 调用 Unlock() 方法释放锁资源时，如果发现有等待唤起的 Goroutine 队列时，则会将队头的 Goroutine 唤起。

队头的 goroutine 被唤起后，会调用 CAS 方法去尝试性的修改 state 状态，如果修改成功，则表示占有锁资源成功。
自旋的条件

还没自旋超过 4 次
多核处理器
GOMAXPROCS > 1
p 上本地 Goroutine 队列为空
首先，如果 mutex 的 state = 0，即没有谁在占有资源，也没有阻塞等待唤起的 goroutine。则会调用 CAS 方法去尝试性占有锁，不做其他动作。

如果不符合 m.state = 0，则进一步判断是否需要自旋。

当不需要自旋又或者自旋后还是得不到资源时，此时会调用 runtime_SemacquireMutex 信号量函数，将当前的 goroutine 阻塞并加入等待唤起队列里。

当有锁资源释放，mutex 在唤起了队头的 goroutine 后，队头 goroutine 会尝试性的占有锁资源，而此时也有可能会和新到来的 goroutine 一起竞争。

当队头 goroutine 一直得不到资源时，则会进入饥饿模式，直接将锁资源交给队头 goroutine，让新来的 goroutine 阻塞并加入到等待队列的队尾里。

对于饥饿模式将会持续到没有阻塞等待唤起的 goroutine 队列时，才会解除。


Unlock 过程

mutex 的 Unlock() 则相对简单。同样的，会先进行快速的解锁，即没有等待唤起的 goroutine，则不需要继续做其他动作。

如果当前是正常模式，则简单的唤起队头 Goroutine。如果是饥饿模式，则会直接将锁交给队头 Goroutine，然后唤起队头 Goroutine，让它继续运行。 
GO读写锁？
RWMutex是一个支持并行读串行写的读写锁。RWMutex具有写操作优先的特点，写操作发生时，仅允许正在执行的读操作执行，后续的读操作都会被阻塞。
RWMutex常用于大量并发读，少量并发写的场景
// 读写互斥锁结构体
type RWMutex struct {
    w           Mutex  // 互斥锁
    writerSem   uint32 // 写锁信号量 //用来唤醒或睡眠goroutine。
    readerSem   uint32 // 读锁信号量 //用来唤醒或睡眠goroutine。
    readerCount int32  // 读锁计数器 当前取读锁goroutine数量
    readerWait  int32  // 获取写锁时需要等待的读锁释放数量 //获取写锁时等待goroutine数量
}

读写锁的读锁可以重入，在已经有读锁的情况下，可以任意加读锁。
在读锁没有全部解锁的情况下，写操作会阻塞直到所有读锁解锁
写锁定的情况下，其他协程的读写都会被阻塞，直到写锁解锁。
写加锁lock()
不管锁是被reader还是writer持有，这个Lock方法会一直阻塞，Unlock用来释放锁的方法
读加锁rlock()
当锁被reader所有的时候，RLock会直接返回，当锁已经被writer所有，RLock会一直阻塞，直到能获取锁，否则就直接返回，RUnlock用来释放锁的方法

数组和切片的区别？
数组是定长
切片的长度可以自动地随着其中元素数量的增长而增长，但不会随着元素数量的减少而减少。
数组是值传递
在每一个切片的底层数据结构中，会包含一个数组，可以被叫做底层数据，而切片就是对底层数组的引用，故而切片类型属于引用类型
type slice struct {
    array unsafe.Pointer // 底层数据的指针
    len   int // 切的长度
    cap   int // 截取底层数据的容量
}
切片扩容？

slice的底层是数组指针。
当append后，slice长度不超过容量cap，新增的元素将直接加在数组中。
当append后，slice长度超过容量cap，将会返回一个新的slice。并且底层指向新的数组
当元素很多应该使用slice，因为go里面都是值传递，而那么slice虽然也是值传递 但是里面存的是指针，不是具体数据
当原切片长度小于1024时，新切片的容量会直接翻倍。而当原切片的容量大于等于1024时，会反复地增加25%，直到新容量超过所需要的容量。
需要注意的是go里面都是值传递 slice内虽然是指针，但是超出容量后会指向新的数组，因此在函数内部追加append的时候不会改变外面的值。反之map不一样。
Go的CSP并发模型？(通讯顺序进程)
GO的map?
go的map底层是hash table 哈希表的实现一般是数组加链表 或者数组加红黑树
buckets负责存数据
hmap结构
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
    count     int
    flags     uint8
    // buckets 的对数 log_2
    B         uint8
    // overflow 的 bucket 近似数
    noverflow uint16
    // 计算 key 的哈希的时候会传入哈希函数
    hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
    buckets    unsafe.Pointer
    // 扩容的时候，buckets 长度会是 oldbuckets 的两倍
    oldbuckets unsafe.Pointer
    // 指示扩容进度，小于此地址的 buckets 迁移完成
    nevacuate  uintptr
    extra *mapextra // optional fields
}

bmap
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
bmap 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置

hash冲突 开放寻址法，链表法。
Map可以边遍历边删除吗
map 并不是一个线程安全的数据结构。同时读写一个 map 是未定义的行为，如果被检测到，会直接 panic。
sync.Map 是线程安全的 map，也可以使用。
无法对 map 的 key 或 value 进行取址

如何比较两个 map 相等？
都为 nil ， 非空 长度相等，指向同一个map对象
相应的 key 指向的 value “深度”相等
因此只能是遍历map 的每个元素，比较元素是否都是深度相等。

map 中的 key 为什么是无序的？

聊聊sync.map?
为什么需要sync.map
因为原本map不是并发安全
并发写入报错。concurrent map writes

低配解决法 
加锁
syncMap结构
type Map struct {
    mu Mutex //互斥锁
    read atomic.Value // readOnly 存读的数据。因为是atomic.Value类型，只读，所以并发是安全的。实际存的是readOnly的数据结构。
    dirty map[interface{}]*entry //包含最新写入的数据。当misses计数达到一定值，将其赋值给read。
    misses int //计数作用。每次从read中读失败，则计数+1。
}

type readOnly struct {
    m  map[interface{}]*entry
    amended bool //Map.dirty的数据和这里的 m 中的数据不一样的时候，为true
}
查询
先从read只读里面查，没有从dirty的新数据查， dirty查询加锁， 从read查没有的时候计数器加1
计数器失败的次数大于 len(dirty) 赋值dirty到read
删除 
read 置nil ， diry直接删除
read的作用是在dirty前头优先度，遇到相同元素的时候为了不穿透到dirty，所以采用标记的方式。 同时正是因为这样的机制+amended的标记，可以保证read找不到&&amended=false的时候，dirty中肯定找不到
增 改
如果m.read存在这个key，并且没有被标记删除，则尝试更新。
总结
内部分为只读map 的新map 读写分离。
Go context?
context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。
并发控制 超时控制
ctx, cancel := context.WithCancel(context.Background())
调用cancal发送通知结束goroutine

type Context interface {
        Deadline() (deadline time.Time, ok bool)//方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。


        Done() <-chan struct{}//方法返回一个只读的chan，类型为struct{}，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过Done方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。


        Err() error


        Value(key interface{}) interface{} //方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。
}
Context接口并不需要我们实现，Go内置已经帮我们实现了2个，我们代码中最开始都是以这两个内置的作为最顶层的partent context，衍生出更多的子Context。
一个是Background，主要用于main函数、初始化以及测试代码中，作为Context这个树结构的最顶层的Context，也就是根Context。

一个是TODO,它目前还不知道具体的使用场景，如果我们不知道该使用什么Context的时候，可以使用这个。


有了如上的根Context，那么是如何衍生更多的子Context的呢？这就要靠context包为我们提供的With系列的函数了。
WithTimeout和WithDeadline基本上一样，这个表示是超时自动取消，是多少时间后自动取消Context的意思。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
withValue不带cancel函数 只用来存储健直对数据
 ctx, cancel := context.WithCancel(context.Background())
        //附加值
valueCtx:=context.WithValue(ctx,key,"【监控1】”)
衍生体都是在context接口上做了扩展
type valueCtx struct {
   Context  // parent Context
   key, val interface{}  // key-value
}
type cancelCtx struct {
    Context
    mu       sync.Mutex            
    done     chan struct{}         
    children map[canceler]struct{}//调用cancel会遍历子节点取消。
    err      error                 
}
变量done表示关闭信号传递；变量children表示当前节点所拥有的子节点，err用于存储错误信息表示任务结束的原因。
什么是byte,什么是rune?
Type byte  = unit8 一个值就是一个ASCII码值。
Type rune =int32 一个值就代表一个Unicode字符
原来是 byte 表示一个字节，rune 表示四个字节，
GO select?
select只能应用于channel的操作，既可以用于channel的数据接收，也可以用于channel的数据发送。

select 随机执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行。一个默认的子句应该总是可运行的。

select具有随机性
select会跳过nil chan
如果没有 default 子句，select 将阻塞，直到某个通信可以运行
应该运用for 配合select 和default 监听 chan
当代码内没有常驻线程， select{}会死锁
select{}. 对应 runtime.block() 阻塞
golang实现select的时候，实际上为每一个case语句定义了一个数据结构，select语句块执行的时候，实际上可以类比成对一个case数组处理的代码块（或者函数），然后程序流程转到选中的case块。

type scase struct {
    c           *hchan         // chan
    elem        unsafe.Pointer // 读或者写的缓冲区地址
    kind        uint16   //case语句的类型，是default、传值写数据(channel <-) 还是  取值读数据(<- channel)
    pc          uintptr // race pc (for race detector / msan)
    releasetime int64
}


在一个select中，所有的case语句会构成一个scase结构体的数组。
[scase, scase, scase]
然后执行select语句实际上就是调用func selectgo()
调用顺序

func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool)
func rselect([]runtimeSelect) (chosen int, recvOK bool)
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)

selectgo函数做了什么

打乱数组顺序，锁住chan, 遍历读取chan, 解锁chan, 判断是否可读可写， 可读写 返回chan 内值。
不可读写走defaul
如果没有可读chan也没有defaul 阻塞当前goroutine，丢到等待队列，下次有chan ready好 唤醒。
sync.Pool?
保存和复用临时对象，减少内存分配，降低 GC 压力。
sync.Pool 的大小是可伸缩的，高负载时会动态扩容，存放在池中的对象如果不活跃了会被自动清理
放进 Pool 中的对象每次 GC 发生时都会被清理掉
不适合做连接池
应用场景

Sync.Once?
Once 可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景。

Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。

sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。

type Once struct {
   done uint32 // 初始值为0表示还未执行过，1表示已经执行过
   m    Mutex 
}

func (o *Once) Do(f func()) {
   // 判断done是否为0，若为0，表示未执行过，调用doSlow()方法初始化
     if atomic.LoadUint32(&o.done) == 0 {
   // Outlined slow-path to allow inlining of the fast-path.
   o.doSlow(f)
     }
}


func (o *Once) doSlow(f func()) {
     o.m.Lock()
     defer o.m.Unlock()
    // 采用双重检测机制 加锁判断done是否为零,是0才会执行。
     if o.done == 0 {
      defer atomic.StoreUint32(&o.done, 1)
      f()
     }
}



Atomic?
sync/atomic包将底层硬件提供的原子操作封装成了 Go 的函数。但这些操作只支持几种基本数据类型，因此为了扩大原子操作的适用范围，Go 语言在 1.4 版本的时候向sync/atomic包中添加了一个新的类型Value。
Mutex由操作系统实现，而atomic包中的原子操作则由底层硬件直接提供支持。在 CPU 实现的指令集里，有一些指令被封装进了atomic包，这些指令在执行的过程中是不允许中断（interrupt）的，因此原子操作可以在lock-free的情况下保证并发安全，并且它的性能也能做到随 CPU 个数的增多而线性扩展。

atomic.Value被设计用来存储任意类型的数据，所以它内部的字段是一个interface{}类型，非常的简单粗暴

type Value struct {
  v interface{}
}

除了Value外，这个文件里还定义了一个ifaceWords类型，这其实是一个空interface (interface{}）的内部表示格式（参见runtime/runtime2.go中eface的定义）。它的作用是将interface{}类型分解，得到其中的两个字段。
type ifaceWords struct {
  typ  unsafe.Pointer //原始类型的指针
  data unsafe.Pointer //数据的指针
}
出于安全考虑，Go 语言并不支持直接操作内存，但它的标准库中又提供一种不安全（不保证向后兼容性） 的指针类型unsafe.Pointer，让程序可以灵活的操作内存。
通过storePointer和loadPointer底层原子函数来存储数据。
应用场景：实现cas 比较交换， 比较交换又可以实现简单的自旋。
Go中两个Nil可能不相等吗
可能
Go的Struct能不能比较?

相同struct类型的可以比较
不同struct类型的不可以比较,编译都不过，类型不匹配

Go的defer原理是什么?
defer变化是由堆分配改为栈分配
循环中的defer是堆分配
type _defer struct {
        siz       int32 参数的大小
        started   bool  是否执行过了
        openDefer bool
        sp        uintptr
        pc        uintptr
        fn        *funcval
        _panic    *_panic
        link      *_defef defer链表，函数执行流程中的defer，会通过 link这个 属性进行串联
}

因为 defer panic 都是绑定在 运行的g上的，所以这里说明一下g中与 defer panic相关的属性

type g struct {
   _panic         *_panic // panic组成的链表
   _defer         *_defer // defer组成的先进后出的链表，同栈
}

综合反编译结果可以看出，defer 关键字首先会调用 runtime.deferproc 定义一个延迟调用对象，然后再函数结束前，调用 runtime.deferreturn 来完成 defer 定义的函数的调用

panic 函数就会调用 runtime.gopanic 来实现相关的逻辑

recover 则调用 runtime.gorecover 来实现 recover 的功能

deferproc函数
根据 defer 关键字后面定义的函数 fn 以及 参数的size，来创建一个延迟执行的 函数，并将这个延迟函数，挂在到当前g的 _defer 的链表上 
这个函数看起来比较简答，通过newproc 获取一个 _defer 的对象，并加入到当前g的 _defer 链表的头部
Deferreturn
return最先执行 返回值赋值，然后defer收尾，最后函数带返回值退出。

Go函数返回局部变量的指针是否安全
这在 Go 中是安全的，Go 编译器将会对每个局部变量进行逃逸分析。如果发现局部变量的作用域超出该函数，则不会将内存分配在栈上，而是分配在堆上。
Goroutine发生了泄漏如何检测
Pprof net/http/pprof
网址ip:port/debug/pprof/打开pprof主页，从上到下依次是类profile信息：


allocs: 内存分配分析


block:同步阻塞分析


cmdline:命令行调用分析


goroutine:goroutine分析


heap:堆内存分析


mutex:锁竞争分析


profile:30s的CPU使用情况分析


threadcreate: 创建新OS线程的堆栈跟踪


trace:当前程序执行的追溯（比如一个get请求的追溯）

临时性泄露，指的是该释放的内存资源没有及时释放，对应的内存资源仍然有机会在更晚些时候被释放，即便如此在内存资源紧张情况下，也会是个问题。这类主要是 string、slice 底层 buffer 的错误共享，导致无用数据对象无法及时释放，或者 defer 函数导致的资源没有及时释放。
永久性泄露，指的是在进程后续生命周期内，泄露的内存都没有机会回收，如 goroutine 内部预期之外的for-loop或者chan select-case导致的无法退出的情况，导致协程栈及引用内存永久泄露问题。


go tool pprof "http://localhost:6060/debug/pprof/heap” 进入堆内存分析
使用top命令查看内存占用最多的函数

(pprof) top10 
Showing nodes accounting for 5.92GB, 100% of 5.92GB total
      flat  flat%   sum%        cum   cum%
    5.92GB   100%   100%     5.92GB   100%  main.main
         0     0%   100%     5.92GB   100%  runtime.main
(pprof) top10
Showing nodes accounting for 5.92GB, 100% of 5.92GB total
      flat  flat%   sum%        cum   cum%
    5.92GB   100%   100%     5.92GB   100%  main.main
         0     0%   100%     5.92GB   100%  runtime.main
(pprof) top10
Showing nodes accounting for 5.92GB, 100% of 5.92GB total
      flat  flat%   sum%        cum   cum%

List main.main 分析具体函数显示第25行

ROUTINE ======================== main.main in /Users/lujunhong/go/src/awesomeProject/main.go
    5.92GB     5.92GB (flat, cum)   100% of Total
         .          .     21:    }()
         .          .     22:
         .          .     23:    tick := time.Tick(time.Second / 100)
         .          .     24:    var buf []byte
         .          .     25:    for range tick {
    5.92GB     5.92GB     26:        buf = append(buf, make([]byte, 1024*1024)...)
         .          .     27:    }
         .          .     28:}



(pprof) traces
Type: inuse_space
Time: Sep 4, 2021 at 12:07am (CST)
-----------+-------------------------------------------------------
     bytes:  5.92GB
    5.92GB   main.main
             runtime.main
-----------+-------------------------------------------------------
     bytes:  4.74GB
         0   main.main
             runtime.main
-----------+-------------------------------------------------------
     bytes:  3.79GB
         0   main.main
             runtime.main
-----------+-------------------------------------


Interface如何实现多态？
不同结构体都去实现interface的方法即可。
进程线程协程
进程可看作为分配资源的基本单位，比如你new出了一块内存，就是操作系统将一块物理内存映射到你的进程地址空间上（进程创建必须分配一个完整的独立地址空间），这块内存就属于这个进程，进程内的所有线程都可以访问这块内存，其他进程就访问不了，其他类型的资源也是同理。
线程作为独立运行和独立调度的基本单位，进而我们可以认为线程是进程的一个执行流，独立执行它自己的程序代码。
线程上下文一般只包含CPU上下文及其他的线程管理信息，线程创建的开销主要取决于为线程堆栈的建立而分配内存的开销，这些开销并不大。
进程：独立的栈空间，独立的堆空间，进程之间调度由os完成。
线程：独立的栈空间，共享堆空间，内核线程之间调度由os完成。
协程：独立的栈空间，共享堆空间，调度由用户自己控制，本质上有点类似于用户级线程，这些用户级线程的调度也是自己实现的。
我们一般将协程理解为用户态轻量级线程，是对内核透明的，也就是系统并不知道有协程的存在，
Golang 有自己的调度器，工作方式基本上是协作式而不是抢占式。 协程切换调度参见gmp
应用程序 保存在计算机磁盘的指令集合
进程是程序资源分配的最小单位，就是一个应用程序的执行过程，负责管理程序的内存资源，io资源，信号等。
线程被包含在进程之中，是进程中的实际运作单位，一个进程内可以包含多个线程，线程是资源调度的最小单位。
我们一般将协程理解为用户态轻量级线程，有独立的栈，调度由用户控制。


操作系统中上下文是运行快照。
三大状态 运行。就绪 阻塞
Io多路复用
虚拟空间：寻址空间，即内核能查找多大内存的范围。
为了保护用户进程不能直接操作内核，操作系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。
文件描述符 ：文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。

缓存I/O
缓存I/O又称为标准I/O，大多数文件系统的默认I/O操作都是缓存I/O。在Linux的缓存I/O机制中，操作系统会将I/O的数据缓存在文件系统的页缓存中，即数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

缓存 I/O 的缺点：
数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

文件句柄：在文件I/O中，要从一个文件读取数据，应用程序首先要调用操作系统函数并传送文件名，并选一个到该文件的路径来打开文件。 该函数取回一个顺序号，即文件句柄（file handle），该文件句柄对于打开的文件是唯一的识别依据。 
多路是指网络连接，复用指的是同一个线程


什么是IO多路复用？

IO 多路复用是一种同步IO模型，实现一个线程可以监视多个文件句柄；
一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作；
没有文件句柄就绪就会阻塞应用程序，交出CPU。

没有IO多路复用机制时，有BIO、NIO两种实现方式，但它们都有一些问题


同步阻塞（BIO）

服务端采用单线程，当 accept 一个请求后，在 recv 或 send 调用阻塞时，将无法 accept 其他请求（必须等上一个请求处理 recv 或 send 完 ）（无法处理并发）

同步非阻塞（NIO）（不知道你要处理哪个连接）
服务器端当 accept 一个请求后，加入 fds 集合，每次轮询一遍 fds 集合 recv (非阻塞)数据，没有数据则立即返回错误，每次轮询所有 fd （包括没有发生读写实际的 fd）会很浪费 CPU。

IO多路复用

服务器端采用单线程通过 select/poll/epoll 等系统调用获取 fd 列表，遍历有事件的 fd 进行 accept/recv/send ，使其能支持更多的并发连接请求。


select

它仅仅知道了，有I/O事件发生了，却并不知道是哪那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。所以select具有O(n)的无差别轮询复杂度，同时处理的流越多，无差别轮询时间就越长。


select缺点

select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理。这样所带来的缺点是：

单个进程所打开的FD是有限制的，通过 FD_SETSIZE 设置，默认1024 ;

每次调用 select，都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大；

需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大

对 socket 扫描时是线性扫描，采用轮询的方法，效率较低（高并发）


poll

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， 但是它没有最大连接数的限制，原因是它是基于链表来存储的.


poll缺点

它没有最大连接数的限制，原因是它是基于链表来存储的，但是同样有缺点：

每次调用 poll ，都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大；
对 socket 扫描是线性扫描，采用轮询的方法，效率较低（高并发时）

epoll

epoll可以理解为event poll，不同于忙轮询和无差别轮询，epoll会把哪个流发生了怎样的I/O事件通知我们。所以我们说epoll实际上是**事件驱动（每个事件关联上fd）**的，此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）


epoll只能工作在 linux 下
消息传递方式

select：内核需要将消息传递到用户空间，都需要内核拷贝动作
poll：同上
epoll：epoll通过内核和用户空间共享一块内存来实现的。

select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。  


Linux下还没有成熟的异步io(指的是内核态), 用户态可以用goroutine。
值接收者和指针接收着的区别？

方法
方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，那么它就变成了方法。接收者可以是值接收者，也可以是指针接收者。
在调用方法的时候，值类型既可以调用值接收者的方法，也可以调用指针接收者的方法；指针类型既可以调用指针接收者的方法，也可以调用值接收者的方法。
也就是说，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。
类型转换和断言的区别
对于类型转换而言，转换前后的两个类型要相互兼容才行
前面说过，因为空接口 interface{} 没有定义任何函数，因此 Go 中所有类型都实现了空接口。当一个函数的形参是 interface{}，那么在函数中，需要对形参进行断言，从而得到它的真实类型。
一个interface{} 里面有Type 代表原始类型。
接口转换的原理？

GO反射？
反射编程是指在程序运行期间，可以访问，检测和修改它本身状态或者行为的一种能力


反射的缺点

由于将部分类型检查工作从编译期推迟到了运行时，使得一些隐藏的问题无法通过编译期发现，提高了代码出现bug的几率，搞不好就会panic
反射出变量的类型需要额外的开销，降低了代码的运行效率
反射的概念和语法比较抽象，过多的使用反射，使得代码难以被其他人读懂，不利于合作与交流
反射的作用
函数需要根据入参来动态的执行不同的行为
不知道传入函数的参数类型，函数需要在运行时处理任意参数对象，这种需要对结构体对象反射。典型应用场景是ORM，gorm示例如下：

```
```

type struct {
gorm.Model
Name string
Age sql.NullInt64
Birthday *time.Time
Email string gorm:"type:varchar(100);unique_index"
Role string gorm:"size:255" // set field size to 255
MemberNumber *string gorm:"unique;not null" // set member number to unique and not null
Num int gorm:"AUTO_INCREMENT" // set num to auto incrementable
Address string gorm:"index:addr" // create index with name addr for address
IgnoreMe int gorm:"-" // ignore this field
}
User := struct{}
db.Find(&users） 

```
```

Golang反射是通过接口来实现的，通过隐式转换，普通的类型被转换成interface类型，这个过程涉及到类型转换的过程，首先从Golang类型转为interface类型, 再从interface类型转换成反射类型, 再从反射类型得到想的类型和值的信息.
在Golang obj转成interface这个过程中, 分2种类型

* 包含方法的interface, 由runtime.iface实现
* 不包含方法的interface, 由runtime.eface实现
reflect. TypeOf 和 reflect.ValueOf 是一个转换器, 完成反射的的最终转换, 得到 reflect.Type, reflect.Value 对象, 得到这2个对象后, 就可以完成反射的准备工作了, 通过 reflect.Type, reflect.Value 这对类型, 可以实现反射的能力.

Golang反射提供的能力

运行时获取对象的类型, 值
创建对象, 执行方法
反射对象转换成Go语言对象
动态修改对象的值



