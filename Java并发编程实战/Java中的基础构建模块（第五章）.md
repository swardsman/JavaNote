## Java中的基础构建模块
Java平台类库包含了丰富的并发基础构建模块，例如线程安全的容器类以及各种用于协调多个相互协作的线程控制流的同步工具类。
### 1.同步容器类
**同步容器类都是线程安全的，但在某些情况下可能需要额外的客户端加锁来保护复合操作。常见的复合操作包括：迭代、跳转（在容器内元素之间）、条件运算（例如“若没有则添加”）。**

**隐式迭代：某些情况下迭代操作会隐藏起来。如下代码中println调用Set的toString方法，然后对Set中的对象进行迭代调用toString方法：**
```
public class HiddenIterator {
	private final Set<Integer> set = new HashSet<Integer>();
    public void addTenThings() {
    	Random r = new Random();
        for (int i = 0;i < 10;i++) add(r.nextInt());
        System.out.println("Debug : " + set);
    }
}
```

### 2.并发容器类
并发容器专门针对多个线程并发访问设计。
**通过使用并发容器代替同步容器，可以极大提高伸缩性并降低风险。**
1. ConcurrentHashMap：替代同步的基于散列Map，在其接口中增加了一些常见符合操作的支持，如“**若没有则添加**”、**替换**、**有条件删除**等。
		ConcurrentHashMap使用了一种粒度更细的加锁机制：分段锁来实现更大程度的共享，能够在并发环境下提高吞吐量。
ConcurrentHashMap不能被加锁来执行独占访问。
**在实际使用中，只有当应用程序需要加锁以进行独占访问时，才应该放弃使用ConcurrentHashMap。**

ConcurrentHashMap中的原子操作：

|方法|说明|
|---|----|
|v putIfAbsent(K key,V value)|仅当k没有相应的映射值时才插入|
|boolean remove(K key,V value)|仅当k被映射到v时才移除|
|boolean replace(K key,V oldValue,V newValue)|仅当k被映射到oldValue时才进行替换|
|V replace(K key,V newValue)|仅当k被映射到某个值时才进行替换|

2. CopyOnWriteArrayLIst：代替同步List，提供了更好的并发性能，并且在迭代期间不需要对容器进行加锁或复制。
“写入时复制（Copy-On-Write）”容器的线程安全性在于，只要正确地发布一个事实不可变对象，那么在访问该对象时就不需要进一步的控制，在每次修改时，都会创建并发布一个新的容器副本，从而实现可变性。
		仅当迭代操作远远多于修改操作时，才应该使用“写入时复制”容器。
        
3. BlockingQueue（生产者----消费者）：相对于Queue，增加了可阻塞的插入和获取等操作。如果队列为空，那么获取元素的操作将一直阻塞，直到队列中出现一个可用的元素，如果队列已满（对于有界队列），那么插入操作将会一直阻塞，直到队列中出现可用空间。
阻塞队列提供了可阻塞的take和put方法，以及支持定时的offer和poll方法。
		在构建高可靠的应用程序时，有界队列是一种强大的资源管理工具：它们能抑制并产生过多的工作项，使应用程序在负荷过载的情况下变得更加健壮。所以应该通过阻塞队列在设计中构建资源管理机制。
        
	- LinkedBlockingQueue
	- ArrayBlockingQueue
	- PriorityBlockingQueue

**串行线程封闭：线程封闭对象只能由一个线程拥有，但可以通过安全地发布该对象来“转移”所有权，在转移所有权后，也只有另一个线程能获得这个对象的访问权限，并且发布对象的线程不会再访问它。**

4. 双端队列
Deque---->BlockingDeque---->ArrayBlockingQueue、LinkedBlockingQueue

### 3.阻塞与中断
线程阻塞的原因：
	- 等待I/O操作结束
	- 等待获得一个锁
	- 等待从sleep方法醒来
	- 等待另一个线程的计算结果

Thread.interrupt()用于中断线程。Thread.interrupt（）方法不会中断一个正在运行的线程。这一方法实际上完成的是，在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态。更确切的说，如果线程被Object.wait， Thread.join和 Thread.sleep三种方法之一阻塞，那么，它将接收到一个中断异常（InterruptedException），从而提早地终结被阻塞状态。

因此，如果线程被上述几种方法阻塞，正确的停止线程方式是设置共享变量，并调用interrupt（）（注意变量应该先设置）。如果线程没有被阻塞，这时调用interrupt（）将不起作用；否则，线程就将得到异常（该线程必须事先预备好处理此状况），接着逃离阻塞状态。在任何一种情况中，最后线程都将检查共享变量然后再停止。

### 4.同步工具类
同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。阻塞队列可以作为同步工具类，其他类型的同步工具类还包括**信号量（Semaphore）**、**栅栏（Barrier）**以及**闭锁（Latch）**。

- 闭锁（Latch）：闭锁是一种同步工具类，可以延迟线程的进度直到其到达终止状态。闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能够通过，当到达结束状态时，这扇门会打开并允许所有的线程通过。当闭锁到达结束状态后，将不会再改变状态，因此这扇门将永远保持打开状态。
**闭锁可以用来确保某些活动直到其他活动都完成后才继续执行**，如：
	1. 确保某个计算在其需要的所有资源都被初始化之后才继续执行
	2. 确保某个服务在其依赖的所有其他服务都已经启动之后才启动
	3. 等待直到某个操作的所有参与者（如在多玩家游戏中的所有玩家）都就绪再继续执行。

Java提供了闭锁的实现：CountDownLatch。它可以使一个或多个线程等待一组时间发生。闭锁状态包括一个计数器，该计数器被初始化为一个正数，表示需要等待的事件数量。countDown()方法递减计数器，表示有一个时间已经发生了；await()方法等待计数器到达0，表示所有需要等待的事件都已经发生，如果计数器非0，await会一直阻塞直到计数器为0，或等待中的线程中断、超时。

- FutureTask
FutureTask实现了Future语义，表示一种抽象的可生成结果的计算。Future.get的行为取决于任务的状态，如果任务已经完成，那么get会立即返回结果，否则get将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。
FutureTask表示的计算是通过Callable来实现的，相当于一种可生成结果的Runnable，并且可以处于以下3种状态：等待运行（Waiting to run)、正在运行（Running）和运行完成（Completed）。
*关于FutureTask的详细信息可以参看blog：
[Java中创建线程的方法](https://my.oschina.net/u/2602561/blog/1554015)*

- 信号量
计数信号量（Counting Semaphore）用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量，计数信号量还可以用来实现某种资源池，或者对容器施加边界。

- 栅栏
栅栏类似于闭锁，它能则色一组线程直到某个事件发生。栅栏与闭锁的关键区别在于：**所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，而栅栏用于等待其他线程**。如果所有线程都达到了栅栏位置，那么栅栏将打开，此时所有的线程都被释放，而栅栏将被重置以便下次使用。
Java中提供CyclicBarrier实现栅栏功能。

	