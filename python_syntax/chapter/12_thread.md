# 多线程

##一、 threading 模块

1. Python 多线程由 `threading` 模块提供

   - 其底层是 `_thread` 模块
   - 如果`_thread` 模块缺失，则使用`dummy_threading` 来模拟它

2. `threading` 模块定义了下面常用的函数：

   - `threading.active_count()` ：返回当前存活的`Thread` 对象的数量（即`threading.enumerate()` 返回的列表的长度）
   - `threading.current_thread()`： 返回本线程对应的`Thread` 对象
     - 如果本线程并不是通过`threading` 模块创建的，则返回一个功能受限的`dummy thread` 对象
   - `threading.get_ident()`：返回本线程的描述符（一个非零的整数值）
     - 线程描述符可能在一个进程中循环使用，因此没有具体的含义
   - `threading.enumerate()`：以列表的形式返回当前存活的所有 `Thread` 对象
     - 这些`Thread` 对象包括： `daemonic thread` ，`dummy thread` ，`main thread`
     - 这些线程对象不包括：已经结束的线程，尚未开始的线程
   - `threading.main_thread()`：返回主线程对象。
     - 通常情况下，主线程就是`python` 解释器启动的那个线程
   - `threading.settrace(func)`：为所有从`threading` 模块启动的线程增加一种锚点函数。
     - `func` 是个可调用对象，它在每个线程的`run()` 函数调用之前被调用。
     - `func` 在底层是被传递给每个线程的`sys.settrace()` 函数，从而完成锚点设置
   - `threading.setprofile(func)`：为所有从`threading` 模块启动的线程增加另一种锚点函数。
     - `func` 是个可调用对象，它在每个线程的`run()` 函数调用之前被调用。
     - `func` 在底层是被传递给每个线程的`sys.setprofile()` 函数，从而完成锚点设置
   - `threading.stack_size([size])` ：设置并返回线程栈的大小（用于接下来新创建线程）
     - 可选的`size` 参数用于设定该线程栈的大小。可以为0，或者一个大于等于 32768 (32KB) 的整数。若未指定`size`，则默认为0
     - 若不支持调整线程栈的大小，则抛出`RuntimeError` 
     - 若设置的线程栈大小无效，则抛出`ValueError` ，并且不调整线程栈大小

3. `threading` 定义了常数`threading.TIMEOUT_MAX` ：对于阻塞函数，最大的超时时间。如果你设置的`timeout` 参数超过了该值，则抛出`OverflowError` 

   > 这是为了防止恶意代码会设置一个非常大的超时时间，导致代码永远阻塞

4. 线程局部数据：通过创建`local` 实例，或者`local` 子类的实例，然后给该实例赋一个属性值，则实现了线程局部数据。

   - 该实例在每个线程中都会有不同的值

   - 其底层实现由`_threading_local` 来实现

   - 其API为：

     ```python
     class threading.local
     ```

     使用方式为：

     ```python
     mydata = threading.local()# 每个线程都会不同
     mydata.x = 1
     ```

## 二、 Thread 

###1.1 基本概念

1.  `Thread` 对象代表一个独立运行的线程。
2.  通常有两种方式来指定线程的执行逻辑：
   - 向`Thread` 类的初始化方法中传入一个可调用对象
   - 在`Thread` 的子类中重写`.run()` 方法。除此之外，子类不应该重写其他的方法（初始化方法除外）
3.  当一个`Thread` 对象被创建之后，调用它的`.start()` 方法就能启动一个线程
   - `.start()`方法将在一个独立的线程空间中自动调用线程的`.run()` 方法。你不应该手动调用`.run()` 方法
   - 当线程启动时，它处于`alive` 状态。你可以通过`.is_alive()` 方法来判断该线程是否存活中
   - 当`.run()` 方法返回，或者`.run()` 方法抛出异常时，线程结束了`alive` 状态
   - 任意线程都可以调用某个`Thread` 的`.join()` 方法。此时调用线程会阻塞直到被`join` 的线程完成
4.  每个线程都有一个`name` 属性，它通过`Thread` 的`name` 参数来设定，并可以通过`name` 属性来修改。
5.  一个线程可以标记为`daemon thread`。
   -  若所有的线程都终止而只剩下`daemon` 线程，则整个进程会退出。
     - 即：当只有主线程和`daemon` 线程，则若主线程退出，则`daemon` 线程也立即被迫终止。
     - 当进程关闭时，`daemon` 线程可能会被粗暴的终止。此时`daemon` 线程占有的资源可能来不及释放。因此如果线程占有了资源，则组好将它设置为非`daemon` 的。
   -  该标记可以通过`Thread` 的`.daemon` 属性来设置，或者通过初始化函数的`daemon` 参数来设置
   -  如果未设置，则线程会继承自父线程的标记 
6.  每个`python` 进程都有一个主线程，它是`Python`程序 最开始执行的那个初始线程。它一定是非`daemon` 的。
7.  有一类线程称作`dummy thread`，它对应于外部线程，其控制权并不由`threading` 模块控制（比如可能由`C` 模块控制）
   - `dummy thread` 对象只有少量的接口，并且总被认为是`alive` 以及`daemonic` 的。并且无法对它调用`.join()` 方法
   - 对于`dummy thread` 对象，无法检测它是否执行结束。因此也就无法对它调用`join()/delete` 等方法。

### 1.2 API 

1. `Thread` 的初始化函数为：

   ```python
   class threading.Thread(group=None, target=None, name=None, args=(), kwargs={}, *, daemon=None)
   ```

   其参数建议都采用关键字参数调用。

   参数为：

   - `group`：保留参数，应该为`None`
   - `target`：一个可调用对象，它会被`run()` 方法自动调用。如果为`None`，则表示线程不做任何事情
   - `name`：一个字符串，为线程名。默认为`'Thread-N'`。其中`'N'` 是小数字。
   - `args`：一个元组，用于`target` 的调用。默认为空元组
   - `kwargs`：一个字典，用于`target` 的调用。默认为空字典
   - `daemon`：用于指定该线程是否`daemon` 线程。
     - 如果为`None`，则继承自父线程
     - 如果非`None`，则直接设定

2. 若`Thread` 的子类重写了`.__init__()` 方法，则必须确保它内部调用了`Thread.__init__()` 方法

3. `Thread` 的方法：

   - `.start()`：启动线程。每个线程最多调用一次。
     - 在内部，它会在一个独立的线程空间中自动调用`.run()` 方法
     - 如果一个线程调用了多次，则会抛出`RuntimeError` 
   - `.run()` ：代表线程要执行的代码逻辑。
     - 你可以在`Thread` 的子类中重写该方法。
     - 该方法默认的执行逻辑是：调用`Thread`初始化方法的`target` 可调用对象（调用参数由 `args,kwargs` 给出）
   - `.join(timeout=None)`：阻塞调用线程，直到等待被`join` 的线程结束（正常结束，或者因抛出异常而结束，或者因超时而结束）。
     - `timeout` 指定超时时间（秒）。如果为`None`，则超时时间为永不超时
     - 由于`.join()` 返回值为`None`，因此你需要在调用`.join()` 之后，再通过`.is_alive()` 方法来确定被`join` 的线程是否结束（因为可能因超时而返回）
     - 可以对一个`Thread` 对象重复调用多次`.join()`
     - 如果对当前线程自己调用`.join()`，则抛出`RuntimeError`，因为会导致死锁。
     - 如果一个线程尚未`start`，对它调用`.join()` 也抛出`RuntimeError`
   - `.getName()/.setName()`：获取、设置线程名字
     - 可以直接通过`.name`属性来实现一样的功能
   - `.is_alive()`：返回线程是否`alive`。
     - 在`.run()` 开始之前，直到`.run()`结束之前，线程都是`alive` 的
   - `.isDaemon()/.setDaemon()`：获取、设置`daemon` 属性
     - 可以直接通过`.daemon` 属性来实现一样的功能

4. `Thread` 属性：

   - `name`：线程的名字。多个线程可能拥有同样的名字，没有什么语法上的意义。
     - 你可以直接修改该属性。其初始值由初始化函数的`name` 参数提供


   - `ident`：线程描述符，一个非零的整数。
     - 如果线程尚未`start`，则为`None`
     - 当一个线程结束时，其描述符被回收利用。因此不同时刻，同一个描述符可能对应不同的线程。
   - `daemon`：返回线程是否是`daemon` 的
     - 该属性必须在`.start()` 之前设置。如果在之后设置，则抛出`RuntimeError` 
     - 由于主线程的`daemon=False`，因此所有子线程的默认`daemon` 都是`False` （从主线程继承而来）

###1.3 限制

1. 在`CPython` 的实现中，由于全局解释锁`GIL` 的存在，`Python` 代码只允许同时存在一个线程。
   - 如果你希望充分利用多核处理器的计算能力，则建议使用多进程机制
   - 但是多线程在 `IO` 占比较高的程序中也能提升程序的性能

##三、 Lock

1. 原子锁是最基础的同步原语。

   - 它有两种状态：`locked`、`unlocked`。原子锁被创建时处于未锁住状态
   - 它有两种方法：
     - `acquire()` ：加锁（也称：获得锁）。
       - 当锁的状态是`unlocked`时，加锁会改变锁的状态为`locked`，并立即返回。
       - 当锁的状态是`locked`时，加锁会阻塞直到锁的状态变为`unlocked` ，然后改变锁的状态为`locked`并返回。
     - `release()` 解锁。
       - 当锁的状态是`locked`时，解锁会改变锁的状态为`unlocked`，并立即返回。
       - 当锁的状态是`unlocked`时，解锁会抛出`RuntimeError` 异常

2. 当有多个线程等待加锁时，只有一个线程会获得锁。至于哪个线程能获得锁，这是不确定的。

3. `LOCK` 的`API`：

   ```python
   class threading.LOCK
   ```

   它没有任何参数。

4. `LOCK` 的方法：

   - `.acquire(blocking=True,timeout=-1)`  ： 加锁。
     - 若`blocking=True`，则当锁不可用时，该方法会被阻塞直到锁可用，并返回`True`
     - 若`blocking=False` ，则当锁不可用时，该方法不会阻塞，而是立即返回并返回 `False`
     - 如果锁可用，则无论`blocking` 是什么值，则该方法都会立即返回并返回`True`
     - 当`timeout` 为一个正的浮点值时，表示超时时间（秒）。如果为`-1` 表示不设超时。
       -  该参数只有当`blocking=True` 时才有效
       -  如果阻塞超时，该方法会返回`False`
   - `.release()` ：释放锁。
     - 该方法可以被任意线程调用（而不仅仅是被获得锁的那个线程调用）
     - 如果在`unlocked` 锁上调用该方法，则抛出`RuntimeError` 异常
     - 该方法没有返回值

##四、 RLock

1.  可重入锁允许同一个线程对同一个锁进行多次加锁操作。它有两个概念：
   - 所属线程：一个可重入锁有两个状态：`unlocked`，`locked`。处于`unlocked` 状态时，没有任何线程用于该锁；处于`locked` 状态时，某个线程拥有它
   - 重入层级：表示拥有该锁的线程对它加了多少次锁（释放的次数相应的抵消掉）。 
     - 对于持有该锁的线程，每当调用一次`acquire` 时，重入层级加1；每当调用一次`release` 时，重入层级减1。当重入层级为0时，释放该锁。
2.  对于可重入锁，一个线程可以多次`acquire`，但是必须要有对应的`release()`
   - `acquire,release` 之间可以相互嵌套
   - 只有最外层的`release` 之后，该锁才真正的被释放掉（处于`unlocked` 状态）。在此之前，锁一直被该线程持有。
3.  可重入锁的API和接口参考`Lock` 

## 五、Condition

1. 条件变量允许一个线程阻塞，直到某个条件（或者事件）发生。

2. 通常条件变量与一个锁相连：

   - 要么人工传入一个锁，要么自动创建一个锁
   - 如果多个条件变量需要共享一个锁，那么必须人工传入锁而不是自动生成
   - 至于这个锁的加锁、释放都由条件变量自动进行

3. 当调用条件变量的`.wait()` 方法时，条件变量必须持有锁。

   - 在条件变量的`.wait()` 方法中，条件变量会首先释放锁，然后等待任意一个线程调用该条件变量的`.notify()` 或者`.notify_all()` 来唤醒它。
   - 一旦条件变量从`.wait()` 中唤醒，则条件变量的`.wait()` 方法会首先获得锁，然后从该方法中返回。
   - 也有可能因为超时而导致等待失败而提前返回。

4. 当调用条件变量的`.nofity()` 方法时，将会唤醒阻塞在条件变量的`.wait()` 方法中的那些线程中的某一个（具体哪个不确定）。

   当调用条件变量的`.nofity_all()` 方法时，将会唤醒阻塞在条件变量的`.wait()` 方法中的全部线程。

   - 注意：条件变量的`.nofity()/.notify_all()` 方法并不会释放锁。因此如果被唤醒的线程无法获得锁，那么它们也会被阻塞在条件变量的`.wait()` 方法中，直到获得锁或者超时。

5. `API`：

   ```python
   class threading.Condition(lock=None)
   ```

   参数：

   - `lock`：如果`lock` 为`None`，则创建一个`RLock` 对象作为底层的锁；否则`lock`  （为`Lock/RLock` 实例）就是底层的锁。

   方法：

   - `.acqurie(*args)`： 获取底层的锁。它直接调用底层锁的`.acquire() ` 方法

   - `.release()`：释放底层的锁。它直接调用底层锁的`.release()` 方法

   - `.wait(timeout=None)`：等待直到`nofity` 发生或者超时。

     - 如果调用时，调用线程尚未获得锁，则抛出`RuntimeError`
     - 当被唤醒时，条件变量的`.wait()` 会重新获得锁并返回
     - 当底层是`RLock` 时，条件变量的`.wait()` 睡眠之前并不是通过`.release()` 方法去释放锁，而是直接调用底层的`API`，无视该`RLock`  被加锁都少次而直接释放。当被唤醒时，该`RLock` 将被加锁，加锁次数与睡眠之前相同
     - 如果成功，则返回`True`；否则返回`False` （若超时，执行失败）

   - `.wait_for(predicate,timeout=None)`： 等待直到`predicate`  为`True`：

     - `predicate` 是个可调用对象，其返回值奖杯视作布尔值

     - 该方法会反复调用条件变量的`.wait()` 方法，直到`predicate` 为 `True` 。`predicate` 校验发生在从条件变量的`wait()` 中唤醒并持有锁的时刻。

     - 该方法的返回值是最后一个调用`predicate` 的值（成功返回）；或者返回 `False` （超时）

       > `predicate` 调用结果不一定是布尔值，可能是正数、字符串等等

   - `.notify(n=1)` ：唤醒 `n` 个等待在该条件变量上的线程。

     - 调用该方法的线程必须持有锁。否则抛出`RuntimeError` 异常
     - 该方法并不会自己释放锁，而是需要手工释放。否则被唤醒的线程奖杯阻塞在获得锁的步骤。
     - 如果有超过`n` 个线程等待被唤醒，则`python` 的实现可能会优化导致唤醒超过`n` 个线程。

   - `notify_all()`： 唤醒所有等待在该条件变量上的线程。

     - 调用该方法的线程必须持有锁。否则抛出`RuntimeError` 异常
     - 该方法并不会自己释放锁，而是需要手工释放。否则被唤醒的线程奖杯阻塞在获得锁的步骤。

##六、信号量

1. 信号量内部维持一个计数。

   - 当调用信号量的`.acquire()` 方法时，计数减1
   - 当调用信号量的`.release()` 方法时，计数加1


   - 计数永远不会小于0。当计数为0时，调用信号量的`.acquire()` 方法会阻塞直到计数大于0。

2. `API`：

   ```python
   class threading.Semaphore(value=1)
   ```

   参数：

   - `value`：信号量的初始值。

   方法：

   - `acquire(blocking=True,timeout=None)`：获取一个信号量。
     - 当信号量的值大于0时，该方法会将信号量的值减1并立即返回 `True`
     - 当信号量等于0时：
       - 当`blocking=False` 时，该方法会立即返回，并返回`False`
       - 当`blocking=True` 时，该方法会阻塞，直到某个线程调用信号量的值大于0。然后改方法会将信号量的值减1并立即返回 `True`
       - 如果阻塞期间，`timeout` 超时，则返回`False`
   - `.release()` 方法：释放一个信号量。它将信号量的值加1

3. 如果有多个线程阻塞在信号量的`.acquire()` 上，则调用一次信号量的`.release()` 只会随机唤醒某一个线程。

4. `class threading.BoundedSemaphore(value=1)`：它是另一种信号量。

   - 该信号量要求任意时刻信号量的值不超过它的初始值。如果超过，则抛出`ValueError` 异常

##七、Event

1.  `Event` 对象代表了某个事件。它是线程间最简单的通信机制。

   - 一个线程发送一个事件，另一个线程等待该事件

2. 一个事件对象其实管理了一个内部标志位。

   - 调用事件对象的`.set()` 方法会让该事件为真
   - 调用事件对象的`.clear()` 方法会让该事件为假
   - 调用事件对象的`.wait()` 方法会让线程阻塞在等待该事件为真

3. `API`：

   ```python
   class threading.Event
   ```

   - 方法：
     - `.is_set()` ：如果内部标志位被设置，则返回`True`，否则返回`False`
     - `.set()` ：设置内部标志位为`True`。所有等待该事件的线程都将被唤醒
     - `.clear()`：设置内部标志位为`False`
     - `.wait(timeout=None)`：调用线程阻塞，直到事件内部标志位为`True`。
       - 如果调用时，事件内部标志位已经为`True`，则立即返回`True` ，表示等待成功
       - 如果阻塞超时，则返回`False`

##八、Timer

1. `Timer` 对象是一个定时器，它在指定事件之后执行某个任务。它是`Thread` 的子类。

2. 当调用定时器对象的`.start()` 方法时，定时器开始启动

   - 在定时器动作发生之前，可以调用定时器对象的`.cancel()` 方法，从而取消定时器
   - 在定时器启动到定时器动作发生之间，有一段等待时间，由`interval` 参数指定。但是等待时间并不是精确的等于`interval` 

3. `API`：

   ```python
   class threading.Timer(interval, function, args=None, kwargs=None)
   ```

   参数：

   - `interval`：等待时间
   - `function`：指定的动作逻辑，是一个可调用对象
   - `args`：传给`function` 的参数。如果为`None`，则默认使用空列表
   - `kwargs`：传给`function` 的关键字参数。如果为`None` ，则使用空字典

   方法：

   - `cancel()`：取消定时器。该方法仅当定时器在等待期间有效

##九、Barrier

1.  `Barrier` 对象用于多个对象的同步。

   - 当有多个线程调用 `Barrier` 对象的`.wait()` 方法时，则所有的线程都被阻塞，直到所有的线程都到达该方法。
     - 它使得所有的线程都在同一个函数上同步
   - 一个`Barrier` 对象可以在同一个线程上使用任意多次。

2. `API`：

   ```python
   class threading.Barrier(parties, action=None, timeout=None)
   ```

   参数：

   - `parties`：指定同步的线程数量。
   - `action`：一个可调用对象。当所有线程阻塞被唤醒的前一刻，`action` 将随机被其中的某个线程执行
     - 如果`action` 抛出异常，则`.wait()` 方法抛出`BrokenBarrierError` 异常
   - `timeout`：指定超时时间。它给出了`.wait()` 方法的默认超时时间。

   方法：

   - `.wait(timeout=None)`：等待所有线程到达该函数点。
     - 一旦所有线程都到达该函数点，则这些线程会被同时解除阻塞。
     - 返回值从 0 到 `parties-1` ，每个线程各不相同。
     - 如果超时，则抛出`BrokenBarrierError` 异常
   - `.reset()`：设置`Barrier` 为初始状态。此时任何阻塞在它的`.wait()` 方法中的线程都会抛出`BrokenBarrierError` 异常
   - `abort()`：将 当前的 `Barrier`  对象设置为`broken state`。此时任何阻塞在它的`.wait()` 方法中的线程都会抛出`BrokenBarrierError` 异常

   属性：

   - `parties`：`Barrier` 对象需要同步的线程的数量
   - `n_waiting`：当前阻塞在`Barrier` 对象的`wait()` 上的线程的数量
   - `broken`：如果当前的`Barrier`  为`broken state` ，则为`True`；否则为`False`

3. `BrokenBarrierError`  是`RuntimeError` 的子类

## 十、 Queue

1. 前面介绍的都是`threading` 模块的。这里的`Queue` 是`queue` 模块的

2. `queue` 模块实现了多个生产者、多个消费者的队列。它常用于多线程环境中，多个线程之间的数据传递。

   - `Queue` 类都对访问进行了加锁机制。

3. `queue` 模块实现了三种队列：

   - `FIFO`：先进先出队列
   - `LIFO`：后进先出队列
   - `priority queue`：优先级队列（使用最大堆排序，每次取出值最低的元素）

4. 先进先出队列：

   ```python
   class queue.Queue(maxsize=0)
   ```

   参数：

   - `maxsize`：一个整数，指定队列的容量。如果该值小于等于0，表示容量为无穷大。
     -  当队列现有的元素个数等于`maxsize` 时，继续插入元素将阻塞直到有空余的空间

5. 后进先出队列：

   ```python
   class queue.LifoQueue(maxsize=0)
   ```

   参数：

   - `maxsize`：一个整数，指定队列的容量。如果该值小于等于0，表示容量为无穷大。
     -  当队列现有的元素个数等于`maxsize` 时，继续插入元素将阻塞直到有空余的空间

6. 优先级队列：

   ```python
   class queue.PriorityQueue(maxsize=0)
   ```

   参数：

   - `maxsize`：一个整数，指定队列的容量。如果该值小于等于0，表示容量为无穷大。
     -  当队列现有的元素个数等于`maxsize` 时，继续插入元素将阻塞直到有空余的空间

   通常加入优先级队列的元素的格式为：`(priority_number,data)`。 每次出队列的是`priority_number` 最小的那个元素。

7. 异常：

   - `queue.Empty`：队列为空的异常。
   - `queue.Full`：队列为满异常。

8. 三种队列都有的方法：

   - `.qsize()`：返回队列的当前大小（它跟容量不是一个概念）。注意它大于0并不能保证接下来获取元素不会阻塞；它小于`maxsize` 并不能保证接下来插入元素不会阻塞

   - `.empty()`： 如果队列为空，则返回`True`。注意它返回`False` 并不能保证接下来获取元素不会阻塞；它它返回`True` 并不能保证接下来插入元素不会阻塞

   - `.full()`：如果队列为满，则返回`True`。 注意它返回`True` 并不能保证接下来获取元素不会阻塞；它它返回`False` 并不能保证接下来插入元素不会阻塞

   - `.put(item,block=True,timeout=None)`：将元素`item` 插入队列。

     - 如果`block=False`，则如果队列允许插入元素，则插入；否则抛出`Full` 异常
     - 如果`block=True`，则：
       - 如果队列允许插入元素，则插入
       - 如果队列已满，则阻塞。如果发生超时，则抛出`Full` 异常。

   - `.put_nowait(item)`：等效于`.put(item,False)`

   - `.get(block=True,timeout=None)`：从队列中取出元素并返回它。

     - 如果`block=False`，则如果队列非空，则取出元素；否则抛出`Empty`异常
     - 如果`block=True`，则：
       - 如果队列非空，则取出元素
       - 如果队列已空，则阻塞。如果发生超时，则抛出`Empty` 异常。

   - `.get_nowait()`：等效于`.get(False)`

   - `.task_done()`：该方法表明一个完整的“插入-取出”流程结束。

     - 该方法由消费者线程调用。对于每个`.get`方法后面，紧跟着调用该方法来通知队列：刚刚被取出的元素已经结束了完整的流程了。
     - 如果调用次数超过了队列的元素的个数，则抛出`ValueError` 异常

   - `.join()`：该方法会阻塞当前线程，直到队列中的所有元素都结束了完整的“插入-取出”流程

     > 队列维护一个 `unfinished` 信号量；调用一次`put` 方法时，该信号量加1；当调用一次 `.task_done()`  时，该信号量减1。



## 十一、其他

1. 所有提供了`.acquire()/.release()` 方法的那些类都可以用于`with` 上下文管理器。
   - 其中包括 `Lock,RLock,Condition,Semaphore,BoundedSemaphore` 类
   - 在进入`with` 块之前，执行`.acquire()` 方法；在退出`with` 块之前，执行`.release()` 方法







