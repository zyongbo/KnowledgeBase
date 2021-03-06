AbstractQueuedSynchronizer是jdk实现锁和其他同步器的基础框架，对于juc包的理解非常重要。
同时AQS的实现为CLH锁的变体。了解CLH锁也至关重要。直接在`AbstractQueuedSynchronizer`中了解所有的方法用途不会很容易，所以这里，只介绍与AQS类相关的CLH锁和AQS的基本`acquire`、`release`方法与Condition,其他的方法通过应用AQS的类来分析。

## CLH

![CLH锁状态转换](https://github.com/TransientWang/KnowledgeBase/blob/master/picture/CLH锁.png)

```java

public abstract class CLHAbslock implements Lock {
    //保存尾部节点的引用，便于将新节点添加到节点尾部
    AtomicReference<Node> tail; 
    //当前线程所持有的节点
    ThreadLocal<Node> curNode;
    //帮助回收节点
    ThreadLocal<Node> preNode;


    public CLHAbslock() {
        //tail的初始化Node就是虚拟头结点，CLH队列需要一个虚拟头结点才能启动
        tail = new AtomicReference<Node>(new Node());
        this.curNode = new ThreadLocal<Node>() {
            @Override
            protected Node initialValue() {
                return new Node(); 
            }
        };
        this.preNode = new ThreadLocal<>();

    }

    class Node {
        volatile AtomicBoolean locked = new AtomicBoolean();
    }
```

我们现在将注意力转向不同类型的队列锁定。上面的代码显示了CLHLock类的字段，构造函数和Node类。此类在Node对象中记录每个线程的状态，该对象具有布尔锁定字段。如果该字段是true，那么相应的线程已获得锁定，或者正在等待锁定。如果该字段为false，则该线程已释放锁定。锁本身表示为Node对象的虚拟链表。我们使用术语“虚拟”，因为列表是隐式的：每个线程通过线程局部pred变量引用其前任。公共尾部字段是最近添加到队列的节点的AtomicReference <Node>。如图7.10所示，为了获取锁，线程将其Node的锁定字段设置为true，表明该线程不是准备释放锁。该线程将getAndSet（）应用于尾部字段，以使其自己的节点成为队列的尾部，同时获取对其前任的Node的引用。然后该线程在前任的锁定字段上旋转，直到前任释放锁。要释放锁定，线程会将其节点的锁定字段设置为false。**然后它重新使用其前驱的Node作为未来锁访问的新节点。它可以这样做，因为此时线程的前驱Node不再被前驱使用，并且线程持有的Node可以由线程的后继者和尾部引用。虽然我们在示例中没有这样做，但它是可能的回收节点**，以便如果有Llocks，并且每个线程一次最多访问一个锁，那么CLHLock类只需要O（L + n）空间，而不是ALock类的O（Ln）。图7.11显示了一个典型的CLHLock执行。当一个线程释放它的锁时，它只使其后继的缓存无效。它提供先到先得的公平性。也许这种锁定算法的唯一缺点是它在无缓存的NUMA体系结构上表现不佳。每个线程都会自旋，等待其前置节点的锁定字段变为false。如果此内存位置是远程的，则性能会受到影响。但是，在缓存一致的体系结构中，这种方法应该可以正常工作。

```java
public class CLHLock extends CLHAbslock {
    @Override
    public void lock() {
        //获取当前线程所持有的节点
        Node node = curNode.get();
        node.locked.set(true);
        //将当前节点添加到tail
        Node pre = tail.getAndSet(node);
        //保存当前节点的前驱节点
        preNode.set(pre);
        
        while (pre.locked.get());

    }

   /**
     * @date 2018/12/17 21:30
     * @return void
     * @Description 当前线程解锁后，当前线程前驱节点就不会再被引用。而当前线程所持有的节点会被后继和尾节点引用。
     * 而在解锁后前驱节点和当前线程持有节点的状态是一样的，所以当前节点完全可以被前驱节点所替换，
     * 当前节点将被JVM回收，也可以不这么做。
     * @Param []
     **/
    @Override
    public void unlock() {
        Node node = curNode.get();
        node.locked.set(false);
        //将当前节点替换为前驱节点，帮助JVM回收当前线程持有的节点
        curNode.set(preNode.get());
    }
}
```

CLHLock class：锁定获取和释放。 最初，尾部字段指的是其锁定字段为false的Node。 Thread A 然后将getAndSet（）应用于尾部字段，以将其QNode插入队列的尾部，同时获取对其前任的QNode的引用。 接下来，B与A相同，将其QNode插入队列的尾部。然后通过将其节点的锁定字段设置为false来释放锁定。 然后它会让自己pred引用的Node，放弃原来持有的Node，以便JVM回收不会再使用的Node。


## AbstractOwnableSynchronizer

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable
```

### java doc

可由线程专有的同步器。 此类为创建可能需要所有权概念的锁和相关同步器提供了基础。 AbstractOwnableSynchronizer类本身不管理或使用此信息。 但是，子类和工具可以使用适当维护的值来帮助控制和监视访问并提供诊断。

```java
/**
 * 独占模式同步的当前所有者
 */
private transient Thread exclusiveOwnerThread;
```

此类只有一个字段，和相应get、set方法。构造函数权限为protected。

## AbstractQueuedSynchronizer

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable 
```

### java doc

提供一个框架，用于实现依赖于先进先出（FIFO）等待队列的阻塞锁和相关同步器（信号量，事件等）。此类旨在成为依赖单个原子int值来表示状态的大多数同步器的有用基础。子类必须定义更改此状态的 protected 方法，并根据要获取或释放的此对象定义该状态的含义。鉴于这些，本类中的其他方法执行所有排队和阻塞机制。子类可以维护其他状态字段，但仅跟踪使用方法getState（），setState（int）和compareAndSetState（int，int）操作的原子更新的int值的同步。
子类应定义为非公共内部帮助程序类，用于实现其封闭类的同步属性。类AbstractQueuedSynchronizer 不实现任何同步接口。相反，它定义了诸如acquireInterruptibly（int）之类的方法，可以通过具体的锁和相关的同步器来适当地调用它们来实现它们的公共方法。

此类支持默认独占模式和共享模式之一或两者。在独占模式下获取时，其他线程尝试获取不能成功。多个线程获取的共享模式可能（但不一定）成功。这个类不会“理解”这些差异，除非在机械意义上，当共享模式获取成功时，下一个等待线程（如果存在）还必须确定它是否也可以获取。在不同模式下等待的线程共享相同的FIFO队列。通常，实现子类仅支持这些模式中的一种，但两者都可以在ReadWriteLock中发挥作用。仅支持独占模式或仅支持共享模式的子类无需定义支持未使用模式的方法。

此类定义了一个嵌套的AbstractQueuedSynchronizer.ConditionObject类，可以通过支持独占模式的子类用作Condition实现，方法isHeldExclusively（）报告是否针对当前线程独占保持同步，使用当前线程调用方法release（int） getState（）值完全释放此对象，并获取（int），给定此保存的状态值，最终将此对象恢复到其先前获取的状态。否则，AbstractQueuedSynchronizer方法不会创建此类条件，因此如果无法满足此约束，请不要使用它。 AbstractQueuedSynchronizer.ConditionObject的行为当然取决于其同步器实现的语义。

此类为内部队列提供检查，检测和监视方法，以及条件对象的类似方法。可以使用AbstractQueuedSynchronizer将它们按需要导出到类中，以实现其同步机制。

此类的序列化仅存储基础原子整数维护状态，因此反序列化对象具有空线程队列。需要可串行化的典型子类将定义一个readObject方法，该方法在反序列化时将其恢复为已知的初始状态。

用法
要使用此类作为同步器的基础，请通过使用getState（），setState（int）和/或compareAndSetState（int，int）检查和/或修改同步状态来重新定义以下方法（如果适用）：

- tryAcquire(int)
- tryRelease(int)
- tryAcquireShared(int)
- tryReleaseShared(int)
- isHeldExclusively()

默认情况下，每个方法都会抛出UnsupportedOperationException。 这些方法的实现必须是内部线程安全的，并且通常应该是短的而不是阻塞的。 定义这些方法是使用此类的唯一受支持的方法。 所有其他方法都被宣布为final，因为它们不能独立变化。
您还可以从AbstractOwnableSynchronizer中找到继承的方法，以便跟踪拥有独占同步器的线程。 建议您使用它们 - 这使监视和诊断工具能够帮助用户确定哪些线程持有锁。

即使此类基于内部FIFO队列，它也不会自动执行FIFO获取策略。 独占同步的核心采用以下形式：

```java
 Acquire:
     while (!tryAcquire(arg)) {
        enqueue thread if it is not already queued;
        possibly block current thread;
     }

 Release:
     if (tryRelease(arg))
        unblock the first queued thread;
```

（共享模式类似，但可能涉及级联信号。）
因为在入队之前调用了获取中的检查，所以新获取的线程可能会先阻塞其他被阻塞和排队的线程。但是，如果需要，您可以通过内部调用一个或多个检查方法来定义tryAcquire和/或tryAcquireShared来禁用插入，从而提供公平的FIFO采集顺序。特别是，如果hasQueuedPredecessors（）（专门设计为由公平同步器使用的方法）返回true，则大多数公平同步器可以定义tryAcquire返回false。其他变化是可能的。

对于默认的驳船（也称为贪婪，放弃和车队避让）策略，吞吐量和可扩展性通常最高。虽然这不能保证公平或无饥饿，但允许先前排队的线程在稍后排队的线程之前重新进行，并且每次重新保留都有一个无偏见的机会成功对抗传入的线程。此外，虽然获取不是通常意义上的“旋转”，但它们可能会在阻塞之前执行tryAcquire的多次调用以及其他计算。当仅短暂地保持独占同步时，这给自旋带来了大部分好处，而没有大部分责任。如果需要，可以通过先前调用获取具有“快速路径”检查的方法来增强此功能，可能预先检查hasContended（）和/或hasQueuedThreads（）仅在同步器可能不会被争用的情况下执行此操作。

该类通过将其使用范围专门化为可依赖于int状态，获取和释放参数的同步器以及内部FIFO等待队列，为同步提供了有效且可扩展的基础。如果这还不够，可以使用原子类，自己的自定义Queue类和LockSupport阻塞支持从较低级别构建同步器。

用法示例
这是一个不可重入的互斥锁类，它使用零值表示解锁状态，1表示锁定状态。 虽然非重入锁并不严格要求记录当前所有者线程，但此类仍然这样做，以便更容易监视使用情况。 它还支持条件并公开一些检测方法：

```java
 class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class
   private static class Sync extends AbstractQueuedSynchronizer {
     // Acquires the lock if state is zero
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (!isHeldExclusively())
         throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Reports whether in locked state
     public boolean isLocked() {
       return getState() != 0;
     }

     public boolean isHeldExclusively() {
       // a data race, but safe due to out-of-thin-air guarantees
       return getExclusiveOwnerThread() == Thread.currentThread();
     }

     // Provides a Condition
     public Condition newCondition() {
       return new ConditionObject();
     }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }
   }

   // The sync object does all the hard work. We just forward to it.
   private final Sync sync = new Sync();

   public void lock()              { sync.acquire(1); }
   public boolean tryLock()        { return sync.tryAcquire(1); }
   public void unlock()            { sync.release(1); }
   public Condition newCondition() { return sync.newCondition(); }
   public boolean isLocked()       { return sync.isLocked(); }
   public boolean isHeldByCurrentThread() {
     return sync.isHeldExclusively();
   }
   public boolean hasQueuedThreads() {
     return sync.hasQueuedThreads();
   }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```

这是一个类似于CountDownLatch的latch类，只是它只需要一个信号来触发。 由于锁存器是非独占的，因此它使用共享的获取和释放方法。

```java
 class BooleanLatch {

   private static class Sync extends AbstractQueuedSynchronizer {
     boolean isSignalled() { return getState() != 0; }

     protected int tryAcquireShared(int ignore) {
       return isSignalled() ? 1 : -1;
     }

     protected boolean tryReleaseShared(int ignore) {
       setState(1);
       return true;
     }
   }

   private final Sync sync = new Sync();
   public boolean isSignalled() { return sync.isSignalled(); }
   public void signal()         { sync.releaseShared(1); }
   public void await() throws InterruptedException {
     sync.acquireSharedInterruptibly(1);
   }
 }
```

 ### Node

等待队列节点类。
等待队列是“CLH”（Craig，Landin和Hagersten）锁定队列的变体。 CLH锁通常用于自旋锁。我们使用它们来park同步器，但是使用相同的基本策略来保存关于其节点的前驱中的线程的一些控制信息。每个节点中的“状态”字段跟踪线程是否阻塞。在其前驱发布时，将发出节点信号。否则队列的每个节点都用作持有单个等待线程的特定通知样式监视器。状态字段不控制线程是否是按钮锁等。线程可能会尝试获取它是否在队列中的第一个。但首先并不能保证成功;它只给予抗争的权利。因此，当前发布的竞争者线程可能需要重新审视。
要排入CLH锁定，您将其原子拼接为新的尾巴。要出列，您只需设置头部字段即可。

```
      +------+  prev +-----+       +-----+
 head |      | <---- |     | <---- |     |  tail
      +------+       +-----+       +-----+
```

插入CLH队列只需要对“尾部”进行单个原子操作，因此存在从未排队到排队的简单原子点划分。同样，出列只涉及更新“头部”。但是，节点需要更多的工作来确定他们的继任者是谁，部分是为了处理由于超时和中断而可能的取消。
“prev”链接（未在原始CLH锁中使用）主要用于处理取消。如果节点被取消，则其后继者（通常）重新链接到未取消的前驱。
我们还使用“next”链接来实现阻塞机制。每个节点的线程ID保存在自己的节点中，因此前驱者通过遍历下一个链接来通知下一个节点，以确定它是哪个线程。后继者的确定必须避免使用新排队节点的竞争来设置其前任的“下一个”字段。必要时，当节点的后继者看起来为空时，通过从原子更新的“尾部”向后检查来解决这个问题。 （或者，换句话说，下一个链接是一个优化，所以我们通常不需要后向扫描。）
 取消为基本算法引入了一些保守性。由于我们必须轮询取消其他节点，我们可能会忽略被取消的节点是在我们前面还是在我们后面。这是通过取消后始终取消park的后继来处理的，这使得他们能够稳定在新的前驱上，除非我们能够确定一位将承担此责任的未经撤销的前驱。
CLH队列需要一个虚拟标头节点才能启动。但是我们不会在构造上创建它们，因为如果没有争用就会浪费精力。相反，构造节点并在第一次争用时设置头尾指针。
等待条件的线程使用相同的节点，但使用其他对队列。条件只需要链接简单（非并发）链接队列中的节点，因为它们仅在完全保持时才被访问。等待时，将节点插入条件队列。根据信号，节点被转移到主队列。状态字段的特殊值用于标记节点所在的队列。

```java
static final class Node {
    /** 标记表示节点正在共享模式中等待 */
    static final Node SHARED = new Node();
    /** 标记表示节点正在独占模式下等待 */
    static final Node EXCLUSIVE = null;

    /** waitStatus值表示线程已取消。 */
    static final int CANCELLED =  1;
    /** waitStatus值表示后继者的线程需要unparking。 */
    static final int SIGNAL    = -1;
    /**waitStatus值表示线程正在等待条件。 */
    static final int CONDITION = -2;
    /**
     * waitStatus值表示下一个acquireShared应无条件传播。
     */
    static final int PROPAGATE = -3;

    /**
     * 状态字段，仅接受值：
     *   SIGNAL:    此节点的后继是（或将很快）被阻塞（通过park），因此当前节点在释放或取消时必须释放其后继。
     *   为了避免竞争，获取方法必须首先指示它们需要信号，然后重试原子获取，然后在失败时停止。
     *   CANCELLED:  由于超时或中断，此节点被取消。 节点永远不会离开这个状态。 
     *               特别是，具有已取消节点的线程永远不会再次阻塞。
     *   CONDITION:  此节点当前处于条件队列中。 在传输之前，它不会用作同步队列节点，此时状态将设置为0.
     *               （此处使用此值与字段的其他用法无关，但可简化机制。）
     *   PROPAGATE:  releaseShared应该传播到其他节点。 在doReleaseShared中设置（仅限头节点）
     *               以确保继续传播，即使其他操作已经介入。
     *   0: 以上都不是
     *
     * 数值以数字方式排列以简化使用。 非负值意味着节点不需要发信号。 
     * 因此，大多数代码不需要检查特定值，仅用于符号。
     *
     * 对于正常的同步节点，该字段初始化为0，对于条件节点，该字段初始化为CONDITION。
     * 它使用CAS（或可能的情况下，无条件的易失性写入）进行修改。
     */
    volatile int waitStatus;

    /**
     * 链接到当前节点/线程依赖的前驱节点以检查waitStatus。 在排队期间分配，并且仅在出列时才被排除（为了GC）。 此外，在取消前驱时，我们在找到未取消的一个时短路，这将永远存在，因为头节点永远不会被取消：节点由于成功获取而变为头结点。
      * 取消的线程永远不会再成功获取，并且线程仅取消自身，而不取消任何其他节点。
     */
    volatile Node prev;

    /**
     * 链接到当前节点/线程在释放时取消驻留的后继节点。 
     * 在排队期间分配，在绕过取消的前驱时进行调整，并在出列时排除（为了GC）。 
     * enq操作直到附加后才分配前驱的next字段，因此看到next字段为null并不一定意味着该节点一定位于队列的末尾。 但是，如果next字段看起来为空，我们可以从尾部扫描prev进行仔细检查。 
     * 已取消节点的下一个字段设置为指向节点本身而不是null，让isOnSyncQueue的判断更容易。
     */
    volatile Node next;

    /**
     * 排队此节点的线程。 在构造时化并在使用后消失。
     */
    volatile Thread thread;

    /**
     * 链接到condition的wait队列的下一个节点，或特殊值SHARED。
     * 因为条件队列只有在保持独占模式时才被访问，所以我们只需要一个简单的链接队列来在节点
     * 等待条件时保存节点。 然后将它们转移到队列中以尝试重新获取锁。 
     * 并且因为condition只能是独占的，所以我们通过使用特殊值来指示共享模式来保存字段。
     */
    Node nextWaiter;

    /**
     * 如果节点在共享模式下等待，则返回true。
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * 返回上一个节点，如果为null则抛出NullPointerException。 
     * 在前驱不能为null时使用。 可以省略空检查，但检查其可以帮助VM。
     *
     * @return 此节点的前驱
     */
    final Node predecessor() {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    /** 建立初始头或SHARED标记。 */
    Node() {}

    /** addWaiter使用的构造函数。 */
    Node(Node nextWaiter) {
        this.nextWaiter = nextWaiter;
        THREAD.set(this, Thread.currentThread());
    }

    /** addConditionWaiter使用的构造方法。 */
    Node(int waitStatus) {
        WAITSTATUS.set(this, waitStatus);
        THREAD.set(this, Thread.currentThread());
    }

    /** CASes waitStatus field. */
    final boolean compareAndSetWaitStatus(int expect, int update) {
        return WAITSTATUS.compareAndSet(this, expect, update);
    }

    /** CASes next field. */
    final boolean compareAndSetNext(Node expect, Node update) {
        return NEXT.compareAndSet(this, expect, update);
    }

    final void setPrevRelaxed(Node p) {
        PREV.set(this, p);
    }

    // VarHandle mechanics
    private static final VarHandle NEXT;
    private static final VarHandle PREV;
    private static final VarHandle THREAD;
    private static final VarHandle WAITSTATUS;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            NEXT = l.findVarHandle(Node.class, "next", Node.class);
            PREV = l.findVarHandle(Node.class, "prev", Node.class);
            THREAD = l.findVarHandle(Node.class, "thread", Thread.class);
            WAITSTATUS = l.findVarHandle(Node.class, "waitStatus", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }
}
```

### 分析

AQS是CLH锁的变体，比CLH的实现要复杂。在这里主要通过ReenTrantLock.lock()和unLock()方法分析AQS行为。

首先关注Node节点的几个常量

`EXCLUSIVE`，这个字段标记当前节点是独占节点。

`SIGNAL`这个常量使用在waitStaus字段上，表示当前节点在解除锁占用后，需要释放后继节点的阻塞状态。

prev，和next字段不是CLH锁中必须有的，在开头的CLH队列中，通过ThreadLocal的`getAndSet()`方法获取前一个节点的引用，形成一个虚拟的链表，而AQS通过显示的引用来获取到前一个节点的状态信息。

next用于唤醒后继线程

`thread`保存当前节点上的线程。

这些字段通过VarHandle来实现CAS操作。

再看一下AQS里的字段：

```java
/**
 * 等待队列的负责人，懒加载。 除初始化外，它仅通过方法setHead进行修改。 注意：如果head存在，则保证其waitStatus不被取消。
 */
private transient volatile Node head;

/**
 *等待队列的尾部，懒加载。 仅通过方法enq修改以添加新的等待节点。
 */
private transient volatile Node tail;

/**
 * 同步状态。
 */
private volatile int state;
```

head节点就是CLH队列的虚拟头结点，它相当于开头的CLH队列中的tail的初始化节点。

tail节点是CLH队列的尾部的引用，帮助新节点添加到队列尾部。state是一个int值，表示当前线程重复获取锁的次数。

#### 获取锁

获取锁行为的方法是acquire：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire`方法以独占模式获取锁，忽略中断。 通过至少调用一次tryAcquire（int）来实现，返回成功。 否则线程排队，可能反复阻塞和解除阻塞，调用tryAcquire（int）直到成功。 

arg代表AQS的state字段。`tryAcquire`需要按照我们的意愿来实现，在AQS类的描述：此方法应查询对象的状态是否允许以独占模式获取它，如果是，则获取它。执行获取的线程始终调用此方法。 如果此方法报告失败，则获取方法可以对线程进行排队（如果它尚未排队），直到通过某个其他线程的释放来发出信号。 

如果调用`acquire`失败则会调用`addWaiter`为当前线程和给定模式（独占或者共享）创建并排队节点。然后`acquireQueued`对于已经在队列中的线程，以独占不间断模式获取。 由条件等待方法使用以及获取。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

```java
Node(Node nextWaiter) {
    this.nextWaiter = nextWaiter;
    THREAD.set(this, Thread.currentThread());
}
```

首先构造了一个新节点新节点的nextWaiter字段设置为Node.EXCLUSIVE==null。此时新节点的waitStatus为默认值0。

然后将新节点入队，如果tail节点为空说明队列需要初始化，调用`initializeSyncQueue`将`head`设置一个新Node,同时将`head`的引用赋值给`tail`节点。现在CLH队列已经被初始化，有了一个虚拟的头结点，可以新插入节点。将新节点的prev字段通过CAS设置为oldTail,将tail通过CAS移动到新节点上。这样新节点就成功插入了队列中。然后将oldTail节点的next指向新节点，这一步不是CAS操作,最后返回尾节点 。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
```

`acquireQueued`会通过自旋的方式不间断的尝试运行`tryAcquire`并且判断当前节点的前驱是否是虚拟头结点。如果前驱节点是头结点并且当前线程持有节点，获取锁成功，则将当前节点替换为头结点，并将前驱节虚拟头结点的next字段设置null，以便于JVM回收。

如果一直获取锁失败在自旋的过程中还会判断，是否应该在获取失败后阻塞当前线程：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * 此置节点已设状态，要求释放信号，以便它可以安全地park。
         */
        return true;
    if (ws > 0) {
        /*
         * 前驱被取消。 跳过前驱并指出重试。
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus一定为0或PROPAGATE。 表示我们需要一个SIGNAL，但不要park。 调用者者需要重试以确保在park前无法获取。
         */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

如果前驱节点的waitStatus== Node.SIGNAL(-1)，则说明之前已经有线程占用了锁。当前节点应该阻塞等待前面的节点唤醒，如果waitStatus > 0 说明前驱节点已经取消，那么可以继续向前遍历到。waitStatus小于0的节点。否则waitStatus就一定为0或者-3，说明前驱已经有节点占用所或者前驱状态为PROPAGATE。应当将前驱节点waitStatus设为SIGNAL（-1）。这样在下次循环时候当前线程就会被阻塞以等待前驱放弃锁。并将当前线程设置为中断状态。（在重入锁中，获取锁的时候自旋两次就会被park）

#### 释放锁

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

`release`以独占模式发布。 如果tryRelease（int）返回true，说明当前线程是占用锁的线程也就是当前线程持有节点是头结点，

释放锁也需要自己实现`tryRelease(arg)`方法，成功释放锁后如果头结点不为空（证明有其他线程排队），则解除后继节点的阻塞状态。为空就说明只有当前线程在使用锁，则不用管后继节点，继续执行。

```java
private void unparkSuccessor(Node node) {
    /*
     * 如果状态为负（即，可能需要信号），则尝试清除signal状态。 
      * It is OK if this fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    /*
     * 线程到unpark是在后继节点，通常只是下一个节点。但是如果下个节点取消或为空时，则从尾部向后移动以找到实际未取消的继任者。
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

如果头节点（独占模式是头结点，也是当前线程持有的节点）的waitStatus<0，则清除头节点的任何状态，然后获取头结点的后继，如果后继为空，说明当前只有一个线程占有锁，如果后继waitStatus>0说明后继节点已经取消则找到下一个没有取消的等待的节点。终于后unpark下一个阻塞的节点。

#### 取消获取节点

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // 跳过取消的前驱
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext是不明显的明显节点。 如果没有，下面的情况将失败，在这种情况下，我们失去了竞争与另一个取消或信号，所以不需要进一步的行动，尽管有可能被取消的节点可能暂时保持可达。
    Node predNext = pred.next;

    // 可以在这里使用无条件写入而不是CAS。 在此原子步骤之后，其他节点可以跳过我们。 之前，我们不受其他线程的干扰。
    node.waitStatus = Node.CANCELLED;

    // 如果我们是尾巴，请自行移除。
    if (node == tail && compareAndSetTail(node, pred)) {
        pred.compareAndSetNext(predNext, null);
    } else {
        // 如果后继者需要signal，请尝试设置pred的下一个链接
        // 所以它会得到一个。否则将其唤醒以进行传播。
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                pred.compareAndSetNext(predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

取消节点的时候，会将当前线程持有的节点的Thread字段置为空。将本节点的waitStatus设置为`Node.CANCELLED`。如果本节点就是尾节点，那么直接删除本节点就可以。如果本节点是头节点，那么直接取消后继节点的阻塞状态就可以。如果不是头尾节点，则将非取消状态的前驱强制变成signal状态，然后链接到当前节点的后继。

## condition

```java
public interface Condition 
```

条件因素将Object监视器方法（wait，notify和notifyAll）分解为不同的对象，以通过将它们与使用任意Lock实现相结合来实现每个对象具有多个等待集的效果。如果Lock替换了synchronized方法和语句的使用，则Condition将替换Object监视方法的使用。
条件（也称为条件队列或条件变量）为一个线程提供暂停执行（“等待”）的手段，直到另一个线程通知某个状态条件现在可能为真。由于对此共享状态信息的访问发生在不同的线程中，因此必须对其进行保护，因此某种形式的锁定与Condition相关联。等待条件提供的关键属性是它以原子方式释放关联的锁并挂起当前线程，就像Object.wait一样。

Condition实例本质上绑定到锁。要获取特定Lock实例的Condition实例，请使用其newCondition（）方法。

举个例子，假设我们有一个有界缓冲区，它支持put和take方法。如果在空缓冲区上尝试获取，则该线程将阻塞，直到某个项可用为止;如果在完整缓冲区上尝试放置，则线程将阻塞，直到空间可用。我们希望继续等待put线程并将线程放在单独的等待集中，以便我们可以使用仅在缓冲区中的项或空间可用时通知单个线程的优化。这可以使用两个Condition实例来实现。

```java
 class BoundedBuffer<E> {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(E x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public E take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       E x = (E) items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   }
 }
```

（ArrayBlockingQueue类提供此功能，因此没有理由实现此示例用法类。）
Condition实现可以提供与Object监视器方法不同的行为和语义，例如保证通知排序，或者在执行通知时不需要保持锁定。如果实现提供了这样的专用语义，那么实现必须记录那些语义。

要注意的是`Condition`实例只是普通的对象，其本身作为一个目标`synchronized`语句，可以有自己的监视器`wait`和`notification`方法调用。  获取`Condition`实例的监视器锁或使用其监视方法与获取与该Condition相关联的`Condition`或使用其waiting和signalling方法没有特定关系。  建议为避免混淆，您永远不会以这种方式使用`Condition`实例，除了可能在自己的实现之内。 

除非另有说明，否则为任何参数传递null值将导致抛出NullPointerException。

实施注意事项:

当等待`Condition`时，允许发生“ *虚假唤醒*  ”，一般来说，作为对底层平台语义的让步。  这对大多数应用程序几乎没有实际的影响，因为`Condition`应该始终在循环中等待，测试正在等待的状态谓词。  一个实现可以免除虚假唤醒的可能性，但建议应用程序员总是假定它们可以发生，因此总是等待循环。 

条件等待（可中断，不可中断和定时）的三种形式在一些平台上的易用性和性能特征可能不同。  特别地，可能难以提供这些特征并保持特定的语义，例如排序保证。  此外，中断线程实际挂起的能力可能并不总是在所有平台上实现。 

因此，不需要一个实现来为所有三种形式的等待定义完全相同的保证或语义，也不需要支持中断线程的实际暂停。 

需要一个实现来清楚地记录每个等待方法提供的语义和保证，并且当一个实现确实支持线程挂起中断时，它必须遵守该接口中定义的中断语义。  

由于中断通常意味着取消，并且检查中断通常是不频繁的，所以实现可以有利于通过正常方法返回来响应中断。  即使可以显示中断发生在另一个可能解除阻塞线程的动作之后，这一点也是如此。 一个实现应该记录这个行为。 

### 分析

Condition与锁一起使用，效果与wait相似，提供线程间通信的方法：

```java
boolean await(long time, TimeUnit unit) throws InterruptedException;
```

`await`导致当前线程等待，直到发出**`signal`或中断**为止。
与此条件关联的锁被原子释放，并且当前线程因线程调度而被禁用，并处于休眠状态，直到发生以下四种情况之一：

- 一些其他线程为此Condition调用signal（）方法，并且当前线程恰好被选为要被唤醒的线程; 
- 其他一些线程为此Condition调用signalAll（）方法; 
- 其他一些线程会中断当前线程，并支持线程挂起中断; 
- 发生“虚假唤醒”。

在所有情况下，在此方法返回之前，当前线程必须重新获取与此条件关联的锁。 当线程返回时，保证保持此锁定。

如果当前线程：

- 在进入此方法时设置其中断状态; 
- 等待时中断并支持线程挂起中断，

然后抛出InterruptedException并清除当前线程的中断状态。 在第一种情况下，未指定在释放锁之前是否发生中断测试。

```java
void signal();	
```

`signal`唤醒一个等待线程。
如果有任何线程在这种情况下等待，则选择一个线程进行唤醒。 然后该线程必须在从await返回之前重新获取锁。

## ConditionObject

```java
public class ConditionObject implements Condition, java.io.Serializable 
```

## 字段

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

只有两个字段代表condition的wait队列如果同一个condition在其他线程中也调用了`await`。将会在此队列中排队。

### 行为分析

### await() 

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

`wait`方法首先会检查当前线程是否被中断，如果中断抛出异常。然后建立一个condition节点加入wait队列中：

```java
private Node addConditionWaiter() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }

    Node node = new Node(Node.CONDITION);

    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

`addConditionWaiter`首先调用`isHeldExclusively`判断当前线程是否是成功占用锁的线程，如果不是则抛出`IllegalMonitorStateException`。然后调用`unlinkCancelledWaiters`将等待队列里面处于取消状态的节点删除。最后新建一个condition节点加入ConditionObject的等待队列里面。

接下来调用`fullyRelease`将AQS的state变量设置为0。

```java
final int fullyRelease(Node node) {
    try {
        int savedState = getState();
        if (release(savedState))
            return savedState;
        throw new IllegalMonitorStateException();
    } catch (Throwable t) {
        node.waitStatus = Node.CANCELLED;
        throw t;
    }
}
```

`fullyRelease`调用`release`将state置0后还会保存下来原值，在唤醒重新加入队列时候用。同时唤醒AQS队列中的下一个等待锁的节点，让其获取锁。接下来调用`isOnSyncQueue`(为什么调用它没分析出来)判断等待队列中的节点是否在AQS等待锁的队列中。

```java
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
   //node.prev可以是非空的，但尚未在队列中，因为将其置于队列中的CAS可能会失败。 所以我们必须从尾部查找以确保它真正成功。 在调用此方法时，它总是接近尾部，除非CAS失败（这不太可能），它将存在，所以我们几乎不会遍历很多。
     
    return findNodeFromTail(node);
}
```

如果Condition队列中的节点不在AQS队列中，则调用`LockSupport.park()`将当前线程阻塞，(放弃监视器)。就像`Object.wait()`一样。需要注意的是park是在一个循环中进行的，循环会判断wait节点是否在等待队列中，这么做的目的与`Object.wait()`一样，是为了**防止线程虚假唤醒的情况**。

到此为止该线程已经成功暂停，下面的代码的作用是在该线程被唤醒之后的行为。

线程被唤醒有两种情况：

1. 在`signal`方法中在将wait节点转入AQS队列后，发现前驱节点已经取消，则会将本线程唤醒，让本线程继续执行，通过`acquireQueued`方法来处理取消的节点。
2. 本线程在wait之前被中断，LockSupport.park()会被唤醒，程序就会继续向下执行并判断这种情况，抛出异常。
3. 线程被`signal`正常唤醒，或在唤醒之后被中断，那么这都属于正常情况，线程在从新获取锁之后则会正常往下执行。



首先线程被唤醒过后调用`checkInterruptWhileWaiting`判断线程是否被中断过，是在`await`期间被中断，还是被唤醒之后被中断，还是没有被中断过：

```java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

如果线程被中断过，则调用`transferAfterCancelledWait`判断具体什么时候被中断过。

```java
final boolean transferAfterCancelledWait(Node node) {
    if (node.compareAndSetWaitStatus(Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    /*
     * 如果我们丢失了一个signal（），那么我们就不能继续它直到完成它的enq（）。 在不完全转移期间取消既罕见又短暂，所以只需自旋。
     */
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

如果该方法返回true,说明在当前线程被唤醒前被中断，并将节点转移到同步队列。如果在当前线程被唤醒之后才被中断，则判断当前节点是否在同步队列中，如果不在说明`signal`出现了异常，自旋让出当前线程。否则返回false,说明当前线程在被signal唤醒后被中断。

```java
private Node enq(Node node) {
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return oldTail;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

`enq`与`addWaiter`相似作用是将ConditionObject队列中的节点加入AQS同步队列中。但它返回的是旧队列尾部，也就是新队列尾部的前驱。

```java
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

最后线程尝试获取锁，`acquireQueued`里面会处理第一种唤醒情况（节点被取消），将ConditionObject等待队列中后面排队的被取消的节点清除。并判断当前线程是否是在被唤醒之前中断过，如果唤醒之前中断过则会抛出异常，如果唤醒之后中断过则会在此调用`Thread.interrupt`再次将自己中断。为什么要再次将自己中断呢，因为程序必须检查线程是够在wait期间被中断过，但是每当调用currentThread().isInterrupted(true)后，就会将当前的中断状态取消（这是可选的）。所以如果唤醒之后线程被中断是合法的情况，程序不需要抛出异常，自然要将自己重新中断。

### signal()

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

`signal`将等待时间最长的线程（如果存在）从此条件的等待队列移动到拥有锁的等待队列。这一点与Object.notify()不同。

`doSignal`删除并传输节点，直到命中未取消的节点或null。 从`signal`方法分离出来是为了鼓励编译器在没有wait节点的时候将方法内联以提高效率。

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```



```java
final boolean transferForSignal(Node node) {
    /*
     * 如果无法更改waitStatus，则该节点已被取消。
     */
    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;

    /*
     * 拼接到队列并尝试设置前任的waitStatus以指示该线程（可能）正在等待。
      * 如果取消或尝试设置waitStatus失败，则唤醒重新同步（在这种情况下，
      * waitStatus可能是暂时且无害的）。
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

`transferForSignal`方法要联系到`await`方法在阻塞线程时的循环代码看。如果节点被取消则继续在Condition队列中遍历下一个。如果将当前节点添加到AQS同步队列后，发现前驱的节点被取消，则直接唤醒所遍历节点的线程。其线程被唤醒后由于节点处于AQS队列中则会跳出循环继续向下执行，在acquireQueue中会进行处理，跳过已取消的节点。直到前面的节点为signal，然后阻塞自己正常等待获取锁。