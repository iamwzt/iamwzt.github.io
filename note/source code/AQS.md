> Provides a framework for implementing blocking locks and 
related synchronizers (semaphores, events, etc) that rely on
first-in-first-out (FIFO) wait queues.

`AQS`依托一个`FIFO`的等待队列，为实现阻塞锁和相关同步器提供了一个框架。
> This class is designed to be a useful basis for most kinds of synchronizers 
that rely on a single atomic int value to represent state.
 
`AQS`以一个原子的int值（`state`）来表示同步器的状态，为大多数同步器提供了一个基础的实现。
> Subclasses must define the protected methods that change this state, and which
define what that state means in terms of this object being acquired or released.  

继承`AQS`的子类必须定义修改状态的受保护方法（`protected`），以及状态对于该实现类`acquired`和`released`的意义。

> Given these, the other methods in this class carry
out all queuing and blocking mechanics. Subclasses can maintain
other state fields, but only the atomically updated int
value manipulated using methods `getState`, `setState` and `compareAndSetState` is tracked with respect
to synchronization.

鉴于以上，`AQS`类中的其他方法能完成所有的排队以及阻塞机制。
子类可以维护所有其他的状态，但只有用以下三个原子更新操作才能实现同步跟踪。
`getState`/`setState`/`compareAndSetState`

> Subclasses should be defined as non-public internal helper classes 
that are used to implement the synchronization properties of their enclosing class.  

子类需被实现为非公有的内部辅助类，借此可以实现其封闭的同步属性。
> Class AbstractQueuedSynchronizer does not implement any
synchronization interface.  Instead it defines methods such as
`acquireInterruptibly` that can be invoked as
appropriate by concrete locks and related synchronizers to
implement their public methods.

`AQS`没有实现任何同步接口，而是定义了诸如`acquireInterruptibly`的方法，并通过具体的锁和同步器来实现其公共方法。

> This class supports either or both a default `exclusive` mode and a `shared` mode. 
When acquired in exclusive mode, attempted acquires by other threads cannot succeed. 
Shared mode acquires by multiple threads may (but need not) succeed. 
This class does not understand these differences except in the mechanical sense 
that when a shared mode acquire succeeds, the next waiting thread (if one exists) 
must also determine whether it can acquire as well. 
Threads waiting in the different modes share the same FIFO queue. 
Usually, implementation subclasses support only one of these modes, 
but both can come into play for example in a ReadWriteLock. 
Subclasses that support only exclusive or only shared modes need not define the methods supporting the unused mode.

`AQS`默认支持`独占`或`共享`中的任一或两种模式。当通过独占模式获取锁时，其它线程就无法成功获取。
多个线程获取共享模式的锁可能（但不一定）成功。

