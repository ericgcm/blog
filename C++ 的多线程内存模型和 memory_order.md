C++ 多线程内存模型在 C++ 11 版本才姗姗来迟，与之相关的语言规范内容就是 std::atomic 库和相关的一些基础设施，比如 std::memory_order，这个主题将介绍 C++ 的多线程内存模型的原理以及 std::memory_order 的应用。
# 1 基本概念

现代 CPU 使用的多核、多线程技术（指一个核心上有多个运算单元和队列，可以同时进行多个并行计算），目的是为了最大化 CPU 的计算能力，这种能力最大化也带来了执行序列的失序和由此带来的内存访问的失序问题。在单核时代，多线程的概念是“宏观上并行，微观上串行”，多线程可以访问相同的 CPU 缓存和同一组寄存器。但是在多核时代，线程可能真的在不同的 CPU 上执行，每个 CPU 都有自己的缓存和寄存器，在一个 CPU 上执行的线程无法访问另一个 CPU 的缓存和寄存器。英特尔 x86/64 处理器也和其他处理器系列一样会在不改变单线程的执行顺序的前提下，根据一定的规则对机器指令的内存交互进行重新排序，特别是允许每个处理器延迟存储并且从不同位置装载数据。于此同时，编译器也会基于自己的规则对代码进行优化，这些优化动作也会导致一些代码的顺序被重排。这种指令的重排，虽然不影响单线程的执行结果，但是会加剧多线程访问共享数据时的数据竞争（Data Race）问题。

本文介绍的 C++ 多线程内存模型、原子变量和 memory_order 本质上不是用来管理多线程之间的数据访问同步，也不是限制多线程之间的执行顺序。很多人会误解 memory_order，将其理解为 mutex，其实无论是功能还是原理，它们俩都不是一回事儿。memory_order 定义的六个模式（或三种数据模型）其实是用来约束同一个线程内的内存访问排序方式的，虽然同一个线程内的代码顺序重排不会影响本线程的执行结果（因为编译器或 CPU 的重排总是要讲规矩的），但是这种代码重排造成的数据访问顺序变化会影响其他线程对所操作的共享数据的访问结果。所以，C++ 的多线程数据模型要解决的问题是如何合理地限制单一线程中的代码执行顺序，使得在不使用锁的情况下，既能最大化利用 CPU 的计算能力，又能保证多线程环境下不会出现逻辑错误。
## 1.1 顺序一致性（Sequential Consistency）

假设有两个线程，分别是线程 A 和线程 B，那么这两个线程的执行情况有三种：第一种是线程 A 先执行，然后再执行线程 B；第二种情况是线程 B 先执行，然后再执行线程 A；第三种情况是线程 A 和线程 B 同时并发执行，即线程 A 的代码序列和线程 B 的代码序列交替执行。尽管可能存在第三种代码交替执行的情况，但是单纯从线程 A 或线程 B 的角度来看，每个线程的代码执行应该是按照代码顺序执行的，这就是朴素的代码顺序一致性模型。

顺序一致性是个好东西，所有操作都按照代码指定的顺序进行，并且所有 CPU（多核）对这些操作的组合结果与全局顺序一致，但是这种严格的排序也限制了现代 CPU 利用硬件进行并行处理的能力，会严重拖累系统的性能。所以，现代 CPU 普遍不支持严格的顺序一致性，来看个例子：
```C++
k=3;    //#1
x=a;    //#2 a 没有使用过，在内存中
y=k;    //#3 k 由于刚刚使用过，可能还在寄存器中暂存
```
在这个例子中，#2 和 #3 的代码并不能保证是顺序执行，编译器可能会让 CPU 在等待数据总线返回 a 的值之前，先执行 y=k 的赋值，因为 k 在之前的代码刚刚访问过，可能还暂存在某个寄存器中，访问更快速。虽然对于单线程来说，这两行代码的顺序并不影响全局结果，但是对于多线程环境来说，如果其他线程也有对 x 和 y 的访问，那么这个线程内的局部优化可能会影响其他线程的结果。

阻止内存访问和指令的重排的方法很多，可以使用编译选项停止此类优化，或者使用预编译指令将不希望被重排的代码分隔开，比如 GCC 可用：
```C++
k=3;
x=a;   
asmvolatile("":::"memory");  //阻止这两行代码顺序改变
y=k;
```
如果是 Visual C++，可以用 _ReadWriteBarrier() 代替。在 CPU 指令层面上，x86/64 的处理器没有提供数据装载和存储屏障之类的设施，但是提供了一些指令可以达到类似的效果，比如 mfence 指令就是一个硬件层面的内存栅栏。除此之外，各种操作系统也提供了类似功能的原语，比如 linux 内核提供了 smp_mb 宏。但是以上这些方法都依赖平台和编译器，使得 lock-free 编程困难重重。C++11 atomic 库的出现使得情况开始好转，它从语言层面提供了一种标准化的方法，使得编写可移植的 lock-free 代码变得更容易了。
## 1.2 happens-before 原则

从程序代码执行的时间顺序看，先执行的操作会 “happens-before” 后执行的操作。对顺序一致性模型来说，前面的代码会 “happens-before” 后面的代码，但是对于现代 CPU 和编译器的并发优化结果来说，代码位置的先后顺序并不一定能保证与执行的先后顺序一致。既然物理上的代码顺序无法保证执行的顺序一致性，那么我们需要定义一个逻辑上的执行顺序一致性，这就是 “happens-before 原则”。如果 A 操作在时间上 “happens-before” B 操作，则认为 A 操作对 B 操作是可见的，即 B 操作能 “看到” A 操作的结果。反过来也一样，如果 A 操作希望对 B 操作是可见的，那么必需满足 A “happens-before” B 这个原则。对于数据并发访问的控制来说，happens-before 原则非常重要，它是判断是否存在数据竞争、线程（数据访问）是否安全的主要依据。

虽然现代 CPU 在执行代码序列的时候是乱序的，但是从程序员的视角看代码的执行必需具备某种顺序确定性才能保证一个功能的正常运行，而满足 happens-before 原则是这种确定性的前提。满足 happens-before 原则需要在软件层面上提供一种比使用 “锁” 更高的语义，用来保证内存访问不仅访问安全，而且逻辑上也符合达成某种任务所需要的操作顺序。C++ 的多线程内存模型就是从语言层面上提供的一种基础设施，用于程序员控制内存访问序列化的行为。
## 1.3 Synchronized-with 关系

Synchronized-with 描述的是一种同步约束关系，简单理解就是如果 A 操作写变量 x，B 操作读变量 x，那么 A 操作 “Synchronized-with” B 操作。同样，如果线程 A 写变量 x，线程 B 读变量 x，则线程 A “Synchronized-with” 线程 B。
# 2 std::memory_order

程序员们为了解决数据并发访问，不得不用各种信号量或锁来同步共享数据的访问，然而加锁是相对接近操作系统的底层原语，并且加锁会增加访问数据的开销，不合适的锁甚至会严重拖累系统的性能。能否从语言的层面提供更高级的语义，既能帮助程序员用更简单的方法保证数据访问的一致性，又能指示编译器更好地对代码进行优化？实际上，很多编程语言都提供了类似的机制和语法规范，C++ 11 终于也提供了这样的语言机制，那就是这个主题要介绍的 std::memory_order。实际上，C++ 的 std::memory_order 并不是具体的方法或对象，它是一组内存访问的约束符号，这些约束符号相互配合，可以实现相应的内存访问模型。

在这一节，我们会通过很多例子代码来解释相关内存访问模型的意义，毕竟一行代码胜过千言万语。关于原子变量（共享变量）std::atomic 的内容在其他主题中已经介绍过，如果不熟悉 std::atomic 的用法可以先阅读相关的主题。另外一点需要注意，应结合编译器的优化思想来理解这里给出的例子代码，所有没有依赖关系的代码，其执行都应理解为是乱序执行的，不是你看到的代码书写顺序。理解这一点对于理解本节的例子和 C++ 的多线程内存模型非常重要。
## 2.1 std::memory_order 定义

C++ 11 的枚举类型 std::memory_order 一共有 6 个值：
```C++
enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
};
```
这六个值对应的内存访问序列化方式可分为三类内存访问模型，分别是：宽松的访问序列化模型、获取/释放语义模型和顺序一致性模型。按照内存访问模型对内存顺序访问控制的强弱排序，如下表所示：
模型	控制强度	枚举值
宽松的访问序列化模型	弱	memory_order_relaxed
获取/释放语义模型	中	memory_order_consume,memory_order_acquire,memory_order_release,memory_order_acq_rel,
顺序一致性模型	强	std::memory_order_seq_cst

需要提一下，在 C++ 20 重新定义了这个枚举：
```C++
enum class memory_order : int {
    relaxed,
    consume,
    acquire,
    release,
    acq_rel,
    seq_cst,

    // LWG-3268
    memory_order_relaxed = relaxed,
    memory_order_consume = consume,
    memory_order_acquire = acquire,
    memory_order_release = release,
    memory_order_acq_rel = acq_rel,
    memory_order_seq_cst = seq_cst
};
```
所以你可以使用 memory_order::relaxed 代替 memory_order_relaxed，不过因为常量表达式的定义，在 C++ 20 使用 memory_order_relaxed 的代码仍然是合法的：
```C++
inline constexpr memory_order memory_order_relaxed = memory_order::relaxed;
inline constexpr memory_order memory_order_consume = memory_order::consume;
inline constexpr memory_order memory_order_acquire = memory_order::acquire;
inline constexpr memory_order memory_order_release = memory_order::release;
inline constexpr memory_order memory_order_acq_rel = memory_order::acq_rel;
inline constexpr memory_order memory_order_seq_cst = memory_order::seq_cst;
```
考虑到避免版本差异的陷阱，使用 memory_order_relaxed 是一个稳妥的方法。

需要C/C++ Linux服务器架构师学习资料加qun812855908获取（资料包括C/C++，Linux，golang技术，Nginx，ZeroMQ，MySQL，Redis，fastdfs，MongoDB，ZK，流媒体，CDN，P2P，K8S，Docker，TCP/IP，协程，DPDK，ffmpeg等），免费分享

## 2.2 宽松的访问序列化模型

memory_order_relaxed 对应的就是宽松的序列化模型，它既可以用于存储数据，也可以用于读取数据。之所以说“宽松”，是因为这种方式只能保证当前的数据访问是原子操作（不会被其他线程的操作打断），但是对内存访问顺序没有任何约束，也就是说对不同的数据的读写可能会被重新排序。用一个例子说明一下：
```C++
int data =42;
std::atomic_booldata_ready(false);
int disorder =0;

void writer_thread()  // 线程1
{
    data =10;        //#1
    data_ready.store(true,std::memory_order_relaxed);   //#2
}

void reader_thread()  // 线程2
{
    while(!data_ready.load(std::memory_order_relaxed));      // #3：对data_ready的读操作
    if(data !=10)  //#4
    {
        disorder++;
    }
}
```
在这个例子中，writer_thread 线程中的 #1 行代码一个是普通的赋值，#2 是带有 memory_order_relaxed 标记的 store 操作，所以它们的执行顺序是不能保证的，也就是说可能被 CPU 或 编译器重排。reader_thread 线程中的 #3 行代码可以保证 happens-before #4 行代码，因为 while 循环的代码语义可以保证，但是 #1 不能保证先于 #4 发生，所以 data 可能不等于 10。需要说明一点，#3 load 操作使用了 memory_order_relaxed，实际上无论 #3 使用什么内存顺序标记，都不改变 #3 happens-before #4 的顺序，因为这个 while 循环会一直测试，直到满足条件才退出。所以代码语义高于任何内存标记，这也是 CPU 或编译器重排代码顺序的规矩。

这就是 memory_order_relaxed 的意义，只能保证当前的 store 是原子操作，但是对内存访问的顺序没有任何约束。上述例子如果希望保证 #1 不被重排到 #2 之后，可以对 #2 的 store 操作使用 memory_order_release，它可以保证在这之前的内存访问不会被重排到它的后面。关于 memory_order_release 的具体含义将在下面的章节中介绍。
```C++
void writer_thread()  // 线程1
{
    data =10;        //#1
    data_ready.store(true,std::memory_order_release);   //#2
}
```
## 2.3 获取/释放语义模型

在这个模型中对应着四个值，分别是 memory_order_consume、memory_order_acquire、memory_order_release 和 memory_order_acq_rel，它们有的适用于读取操作，有的适用于存储操作，有的既能用于读取操作，也能用于存储操作。用这些描述符，配合相应的内存读写操作，可以实现相对严格一点的内存访问顺序控制。它们通常不是单独使用，而是配合使用，比如 Release-Acquire 顺序、Release-Consume 顺序以及 “读-修改-写”顺序。

### 2.3.1 memory_order_release

一个对原子变量 A 的存储（store）操作如果施加了 memory_order_release 标记，它对内存操作的影响是：当前线程中这个存储操作之前的内存读、写操作都不可以被重排在这个存储操作之后；其他线程中如果对原子变量 A 施加了带有 memory_order_acquire 标记的读取（load）操作，则当前线程中这个存储操作之前的其他内存读、写操作对这个线程可见；其他线程中如果有对这个原子变量 A 施加了带有 memory_order_consume 标记的操作，则当前线程中所有原子变量 A 所依赖的内存写操作对其他线程可见（没有依赖关系的内存操作就不能保证顺序）。

上述描述有点抽象，不好理解，但是不要紧，下面节结合 memory_order_acquire 和 memory_order_consume 介绍 Release-Acquire 顺序和 Release-Consume 顺序的时候会用具体的例子详细说明。这里只需记住一点：对当前线程来说，当前带有 memory_order_release 标记的原子操作之前的所有内存读写，在一定条件下不会被重排到这个原子操作的后面，至于对其他线程的影响都是这个对当前线程的约束所产生的副作用。很多资料对 memory_order_release 标记的理解有误，他们认为 memory_order_release 标记之前的所有内存读写都不会被重排在这个标记的操作之后，其实是不对的，这里要强调是有条件的，这个条件就是与之配对使用的是 memory_order_acquire 还是 memory_order_consume。

### 2.3.2 memory_order_acquire

一个内存读取操作如果施加了 memory_order_acquire 标记，它对内存操作的影响是：对当前线程来说，这个读取操作后面的所有的内存读和写操作都不能被重排到这个读取操作之前；其他线程中如果有对当前读取的原子变量施加了带有 memory_order_release 标记的写操作，则这个原子变量写操作之前的内存读、写操作对当前线程都是可见的。这里的可见请结合 happens-before 原则理解，即那些内存读写操作会确保完成，不会被重新排序。

如果线程 A 中的一个原子变量的存储操作被标记为 memory_order_release，线程 B 中来自同一原子变量的读取操作被标记为memory_order_acquire，那么从线程 A 的角度来看，在原子变量存储之前发生的所有内存写入（包括非原子变量和使用 memory_order_relaxed 标记的原子变量）在线程 B 中都会产生作用。也就是说，一旦原子读取完成，线程 B 就可以保证看到线程 A 写入内存的所有内容。这里说的同步只在 release 和 acquire 相同原子变量的线程之间建立，其他线程可能看到与同步线程不同的内存访问顺序。

看一个典型的“生产者-消费者”例子：
```C++
std::atomic<std::string*> ptr;
int data;

void producer()
{
    std::string* p  =newstd::string("Hello");   //#1
    data =42;                                    //#2
    ptr.store(p,std::memory_order_release);      //#3
}

void consumer()
{
    std::string* p2;
    while(!(p2 = ptr.load(std::memory_order_acquire)));      //#4
    assert(*p2 =="Hello");                  //#5
    assert(data ==42);                      //#6
}

int main()
{
    std::threadt1(producer);
    std::threadt2(consumer);
    t1.join(); t2.join();
}
```
在这个例子代码中，原子变量 ptr 的 store 操作使用了 memory_order_release 标记，这意味着在 producer 线程中，#1 和 #2 的内存操作不会被重排到这个 store 操作之后。在 consumer 线程中，对同一原子变量 ptr 的 load 操作使用了 memory_order_acquire，这意味着当其确定取到不为 null 的值的时候，#1 和 #2 对 consumer 线程是可见的（即 #1 和 #2 一定会先执行），所以 #5 和 #6 的断言能保证成立。这个例子再次说明了 C++ 的 std::memory_order 不是用来做线程同步的，它的意义仅仅在于当 consumer 线程中对原子变量 ptr 取到非 null 的值的时候，producer 线程中使用的 memory_order_release 标记能确保 ptr 在存储为非 null 值之前 #1 和 #2 的赋值操作已经完成（因为不会被重排到这个存储操作之后）。

### 2.3.3 memory_order_consume

一个内存读取操作如果施加了 memory_order_consume 标记，它对内存操作的影响是：当前线程中任何与这个读取操作有依赖关系的读写操作都不会被重排到当前读取操作之前；其他线程中如果对这个读取操作的原子变量施加了带有 memory_order_release 标记的操作，则那个线程中所有与这个原子变量有依赖关系的写操作对当前线程是可见的。在大多数平台上，这个标记只影响编译器的优化结果（代码重排），不影响 CPU 的指令重排。

依赖关系非常容易理解，我们举个例子说明一下，如下代码中，变量 a 的值不依赖变量 b，但是变量 b 的值依赖于变量 a 的值，这就是 b 依赖 a 的关系。
```C++
int a =1;
int b = a +1;
再看一个例子：
std::atomic<std::string*> ptr;
int data;

std::string* p  =newstd::string("Hello");
data =42;                                   
ptr.store(p,std::memory_order_release);
```
在这个例子中，原子变量 ptr 依赖 p，但是不依赖 data，data 和 p 互相不依赖。

了解了依赖关系之后，回过头继续理解 memory_order_consume 标记的意义。如果线程 A 中的原子变量 store 操作被标记为 memory_order_release，线程 B 中读取同一个原子变量的 load 操作被标记为 memory_order_consume，则从线程 A 的角度来看，在原子变量存储之前发生的所有内存写入（包括非原子变量和使用 memory_order_relaxed 标记的原子变量）中，只有这个原子变量有依赖关系的内存读写才会保证不被重排到这个 store 操作之后。也就是说，当线程 B 使用带 memory_order_consume 标记的 load 操作时，线程 A 中只有与这个原子变量有依赖关系的内存读写操作才保证不会被重排到 store 操作之后。与之对应的是，如果线程 B 使用带 memory_order_acquire 标记的 load 操作时，线程 A 中所有在 store 之前的所有内存读写操作都保证不会被重排到 store 操作之后。

同样用上一节的例子，只是将 consumer 线程的 load 操作换成 memory_order_consume 标记，对比一下它们的差异：
```C++
std::atomic<std::string*> ptr;
int data;

void producer()
{
    std::string* p  =newstd::string("Hello");   //#1
    data =42;                                    //#2
    ptr.store(p,std::memory_order_release);      //#3
}

void consumer()
{
    std::string* p2;
    while(!(p2 = ptr.load(std::memory_order_consume)));      //#4
    assert(*p2 =="Hello");                  //#5
    assert(data ==42);                      //#6
}
```
producer 线程中的代码没有变化，但是 consumer 线程对同一个原子变量的 load 操作换成 memory_order_consume 标记，使得 producer 线程的行为发生明显变化：#1 的 p 与 原子变量 ptr 有依赖关系，所以不会被重排到 #3 之后，但是 data 与 ptr 没有依赖关系，所以 data = 42 这行代码可能会被重排到 #3 之后。这个变化使得 #6 的断言可能会出现失败的情况。

那么什么时候用 memory_order_consume，什么时候用 memory_order_acquire 呢？当我们对一个原子变量使用带 memory_order_acquire 标记的 load 的时候，对那些使用 memory_order_release 标记 store 这个原子变量的线程来说，这些线程中在 store 之前的所有内存操作都不能被重排到 store 之后，这将严重限制 CPU 和编译器优化代码执行的能力。所以，当确定只需对某个变量限制访问顺序的时候，应尽量使用 memory_order_consume，减少代码重排的限制，对性能有好处。

### 2.3.4 memory_order_acq_rel

对单独的 load 或 store 操作施加内存顺序标记会对特定的 load 或 store 产生影响，但是对于 atomic_exchange，或者 compare_exchange 这样在一个原子操作中既有 load ，又有 store 的操作，如何对这一个原子操作中的两个动作施加标记呢？这就要用到 memory_order_acq_rel。对于施加了 memory_order_acq_rel 标记的原子操作，这个标记对当前线程的影响是：当前线程中此操作之前或之后的内存读写都不能被重新排序。对其他线程的影响是：在修改之前，如果其他线程中对这个原子变量施加了 memory_order_release 标记的 store 操作，则那个线程中所有 store 之前的内存写入操作对当前线程可见（当前线程做修改动作之间，能看到那个线程的写入结果）；如果其他线程中对这个原子变量施加了 memory_order_acquire 标记的 load 操作，则当前线程的修改操作对那个线程也是可见的。

简单理解，这个标记相当于对当前原子操作中的 load 操作施加了 memory_order_acquire 标记，对 store 操作施加了 memory_order_release 标记。于此同时，当前线程中这个操作之前的内存读写不能被重排到这个操作之后，这个操作之后的内存读写也不能被重排到这个操作之前。

通过一个 3 线程的例子理解这个标记：
```C++
std::vector<int> data;
std::atomic<int> flag ={0};

void thread_1()
{
    data.push_back(42);                        // #1
    flag.store(1,std::memory_order_release);  //#2
}

void thread_2()
{
    int expected=1;  // #3
    while(!flag.compare_exchange_strong(expected,2,std::memory_order_acq_rel))//#4
    {
        expected =1;
    }
}

void thread_3()
{
    while(flag.load(std::memory_order_acquire)<2);//#5
    assert(data.at(0)==42);                         //#6
}
```
thread_2 线程中对原子变量 flag 的 compare_exchange 操作使用了 memory_order_acq_rel 标记，意味着这个线程中 #3 不会被重排到 #4 之后，也就是说，当 compare_exchange 操作发生的时候，能确保 expected 的值是 1，使得这个 compare_exchange 操作能够完成将 flag 替换成 2 的动作。thread_1 线程中对 flag 使用了带 memory_order_release 标记的 store，这意味着当 thread_2 线程中取 flag 的值得时候，#1 已经完成（不会被重排到 #2 之后）。当 thread_2 线程 compare_exchange 操作将 2 写入 flag 的时候，thread_3 线程中带 memory_order_acquire 标记的 load 操作能看到 #4 之前的内存写入，自然也包括 #1 的内存写入，所以 #6 的断言始终是成立的。
## 2.4 顺序一致性模型

顺序一致性模型对应的约束符号是 memory_order_seq_cst，这个模型对于内存访问顺序的一致性控制是最强的，类似于很容易理解的互斥锁模式，先得到锁的先访问。对原子变量的访问使用 memory_order_seq_cst 标记意味着：如果是读取数据的 load 操作，它相当于对 load 施加 memory_order_acquire 标记，如果是存储数据的 store 操作，它相当于对 store 施加 memory_order_release 标记。如果是 “读取-修改” 操作，它相当于施加了 memory_order_acq_rel 标记。对所有线程来说，这个标记会对所有使用此标记的原子变量进行同步，使得线程看到的内存操作的顺序都是一样的。前面介绍的 Release-Acquire 顺序或 Release-Consume 顺序都是用来同步一个原子变量的访问顺序，这个 memory_order_seq_cst 则是同步所有原子变量的访问顺序，就像是将所有原子变量的操作放在一个线程中顺序执行一样。

顺序一致性模型容易理解，std::atomic 的操作都使用 memory_order_seq_cst 作为默认值。如果你不确定使用何种内存访问模型，用 memory_order_seq_cst 能确保不出错。
## 2.5 总结

宽松的访问序列化模型对内存访问的顺序一致性控制是最弱的，约等于没有控制。这个模型只能保证每个读写动作是不可被打断的原子操作，但是一个读数据的线程可能在写数据的线程写入之前读到旧的数据，也可能在写入之后读到新的数据，完全是随机的。顺序一致性模型虽然容易理解，对内存访问的顺序一致性要求最高，但是严格遵守顺序一致性会给编译器的优化带来很大的限制，对现代 CPU 的处理能力也是一种浪费。
# 3 后记

很难理解，对吧？为什么 C++ 就不能把内存模型设计的简单一点呢？这当然体现的 C++ 设计的一贯“恶趣味”，就是为了效率可以牺牲易用性。C++ 的多线程内存模型为了提供足够的灵活性和性能，将内存顺序一致性的原理甚至是实现细节都暴露给了程序员，给高手足够的发挥空间，也让新手一脸茫然。但是一定要搞成这么高深的样子吗？有时候也是被逼的，以下内容摘自《Thriving in a Crowded and Changing World: C++ 2006-2020》：
```我们知道 Java 有一个很好的模型，我们曾希望采用它。令我感到好笑的是，来自英特尔和 IBM 的代表坚定地否定这个想法，他们指出，
如果在 C++ 中采用 Java 的内存模型，那么我们将使所有 Java 虚拟机的性能减慢至少两倍。因此，为了保持 Java 的性能，我们不得不
为 C++ 采用一个复杂的多的模型。可以想见而讽刺的是，C++ 此后因为有一个比 Java 更复杂的内存模型而受到批评
```
最后要说明一下，x86/64 平台是一个强序内存模型（Strong Ordering）平台，这意味着对于原子变量默认是执行“获取/释放（ acquire-release）”语义，所以一些资料里的数据竞争的例子在 x86/64 CPU 上是跑不出来的，比如本文 2.2 节的例子，还有 C++ 官方资料上引用的这个例子：
```C++
std::atomic<bool> x,y;
std::atomic<int> z;

void write_x_then_y()
{
     x.store(true,std::memory_order_relaxed);
     y.store(true,std::memory_order_relaxed);
}

void read_y_then_x()
{
    while(!y.load(std::memory_order_relaxed));  
    if(x.load(std::memory_order_relaxed))
         ++z;
}
```
理论上说，对这两个线程重复跑，有一定概率能跑出 z = 0 的结果，因为 write_x_then_y 线程中 x 和 y 的 store 操作用的是 memory_order_relaxed，这个不足以保证它们的顺序。但是实际上在 intel 的 CPU 上跑不出 z = 0 的结果，如果使用嵌入式系统常用的 PowerPC 和 ARM CPU，应该就可以了，因为它们是弱序内存模型（Weak Ordering）系统。

最后，关于 C++ 的 memory order，总结一下：
```markdown
    使用了标准同步原语（比如自旋锁、信号量）的代码不需要显式的使用 memory order，因为这些原语中已经存在必要的顺序约束，
只有一些因性能要求需要避免使用同步原语的复杂代码才需要用到 memory order。对内存的读写访问代码如果都在同一个线程中，也不
需要考虑使用 memory order，因为它们在同一时刻只会在同一个 CPU 上执行。只有在不同线程（它们可能同时在不同的 CPU 上并行执行）
中互相访问对方线程的数据的场合，才需要考虑使用 memory order。强内存模型系统实际上隐含实现了“获取/释放（acquire-release）”语义。
```
