# 高级并行

## 一、调度器

1. 调度器由`sched` 模块提供，提供了通用的事件调度器`scheduler`

   - `scheduler` 是线程安全的，可以用于多线程环境
   - 每个定时器都在同一个线程中完成。如果你需要多线程执行，则可以考虑使用多个定时器，并安排到多个线程中。

2. `API`：

   ```python
   class sched.scheduler(timefunc=time.monotonic, delayfunc=time.sleep)
   ```

   参数：

   - `timefunc`：一个可调用对象（它不带任何参数），其调用的结果是一个数字（表示时间，可以为任意单位），表示调度器的时间基准。

     - 默认采用`time.monotonic`，它返回的是系统启动以来的流逝时间。如果不可用，则使用`time.time`，它返回的是当前系统的时间戳（从1970年以来的秒数）

   - `delayfunc`：一个可调用对象

     - 它带一个参数，表示延迟时间。其单位与`timefunc` 的返回结果一致。
     - 如果定时器中没有任何事件已到期，则定时器会调用`delayfunc`，参数为最近的一个快到期的事件的定时时刻距离当前的距离。
     - 如果定时器中已经有事件到期，则定时器会先调度事件，然后调用`delayfunc(0)` 。参数为0，表示让其它线程有机会执行。

     > 无论有没有事件到期，每一次事件队列的查询结束时，都调用`delayfunc` 。这是因为调度器的调度是在单个线程的。如果没有 `delayfunc` ，则查询的循环会一直持有CPU，使得别的线程没有机会执行。

3. 方法：

   - `.enterabs(time, priority, action, argument=(), kwargs={})`：根据绝对时间来调度一个新的事件。

     - `time`：一个数字，它的单位与`timefunc` 返回的一致，表示事件被调度的绝对时间。

     - `priority`：一个数字，表示事件的优先级。数字比较大的优先级较高。

       > 优先级用于：当同一时刻有两个事件需要被调度，则优先级高的先执行

     - `action`：一个可调用的对象，表示事件的执行逻辑。其位置参数为`argument`，关键字参数为`kwargs`。调度器将调用`action(*argument,**kwargs)`

     - 返回一个事件对象。

   - `.enter(delay, priority, action, argument=(), kwargs={})`：根据相对延迟来调度一个新的时间。

     - `delay`：一个数字，它的单位与`timefunc` 返回的一致，表示事件被调度的相对延时。
     - 其他参数与返回结果，参考 `enterabs` 方法

   - `.cancel(event)`： 取消一个事件。

     - 事件`event` 是由`enterabs/enter`  方法返回的
     - 如果`event` 并不在调度器当前的事件队列中，则抛出`ValueError` 异常

   - `.empty()`： 如果调度器当前的事件队列为空，则返回`True`

   - `.run(blocking=True)`： 调度所有的事件。

     - 该方法会串行的对事件队列中的每一个事件进行调度，直到所有的事件都完成

       > 它并没有让每个事件分配一个线程。而是所有的线程都在调度器本地的线程执行。属于串行执行，而不是并行执行。

     - 如果`blocking=False`，则：如果有事件到期，则执行它。返回最近的一个快到期的事件的定时时刻距离当前的距离。

     - `action`（事件逻辑） 和`delayfunc`（调度逻辑）可能会抛出异常。

       - 即使抛出异常，调度器也会维持状态的一致性，并抛出这个异常
       - 如果是`action` 抛出异常，则该事件以后不会再被调度了

4. 属性：

   - `queue`：一个只读的属性，返回一个事件列表（元素顺序与它们被执行的顺序相同，但是可能与插入的顺序不同，因为要考虑优先级）
     - 每个事件都是一个`named tuple`，元素为 `(time,priority,action,argument,kwargs)`

5. 调度器对象有几点要注意：

   - 调度器对象，及其调度的事件都在同一个线程中运行，而不是多线程并行执行
   - 给定一个事件的调度时刻，该事件并不一定在那个时刻得到执行。
     - 调度器采取轮训的策略，每轮只会选取目前最先到期的事件执行。如果该事件非常耗时，可能会使得大量的事件在这个过程中到期。

## 二、futures

1. `concurrent.futures`  模块提供了更高级别的异步执行接口。

   - 可以通过`ThreadPoolExecutor` 来异步执行，它使用多线程
   - 可以通过`ProcessPoolExecutor` 来异步执行，它使用多进程

   > 二者的接口都是一致的，都是`Executor` 的子类

### 2.1 Executor

1.  `Executor` 是个抽象的基类，它提供了异步调用的一些接口

   - 你应该使用它的子类，而不是直接使用它

2. API：

   ```python
   class concurrent.futures.Executor
   ```

3. 方法：

   - `.submit(fn,*args,**kwargs)`：异步调用一个事件`fn`，调用方式为 `fn(*args,**kwargs)`。返回一个`Future` 对象来代表执行的结果

   - `map(func, *iterables, timeout=None, chunksize=1)`：内置 `map` 函数的异步调用形式。返回一个迭代器对象

     - 对于返回的迭代器对象，每次对它迭代（即：调用其 `.__next__()` 方法）时，如果发生超时（由`timeout` 参数给出，是相对于`map` 方法被调用的秒数），则抛出`concurrent.futures.TimeoutError` 异常
     - 如果调用抛出了异常，则对返回的结果迭代时该异常会被重新抛出
     - 当使用`ProcessPoolExecutor` 时，该方法会把数据划分成小块，然后分配给每个独立的子进程。每块大小由`chunksize` 决定（默认为1）。对于`ThreadPoolExecutor`，该参数无效。

   - `.shutdown(wait=True)`：在当前正在执行的工作子线程/工作子进程完成执行之后，关闭该`executor` （释放资源）

     - 在调用了`.shutdown` 之后，如果后面又调用了`.submit()` 或者 `.map()` 方法，则抛出`RuntimeError` 

     - 如果`wait=True`，则当前的`executor` 会等待正在执行的工作子线程/工作子进程执行结束。然后释放当前的`executor` 分配的资源。然后该方法返回。

     - 如果`wait=False`，则该方法立即返回。而当前`executor` 分配的资源会等待正在执行的工作子线程/工作子进程执行结束之后自动被释放

       > 无论`wait` 参数如何，当前`executor` 分配的资源都会在正在执行的工作子线程/工作子进程执行结束之后自动被释放

4. `Executor` 支持上下文管理器协议。

   - 其 `__enter__` 方法返回`executor` 自己
   - 其`__exit__` 方法自动调用 `.shutdown(wait=True)` 方法

   ```python
   with ThreadPoolExecutor(max_workers=1) as executor:
       future = executor.submit(pow, 323, 1235)
       print(future.result())
   ```

### 2.2 ThreadPoolExecutor

1.  `ThreadPoolExecutor` 是 `Executor` 的子类，它使用线程池来执行异步调用

2. `API`：

   ```python
   class concurrent.futures.ThreadPoolExecutor(max_workers=None, thread_name_prefix='')
   ```

   参数：

   - `max_workers`：指定线程池的容量（最多`max_workers` 个工作线程）。
     - 如果为`None`，则默认使用CPU数量的5倍。
     - `thread_name_prefix`：`python 3.6` 新增。用于设定线程池的工作线程名的前缀，方便调试。

3. 如果在异步调用的任务中，需要使用到另一个异步调用任务的结果，则可能会死锁

   - 如果两个`executor` 的任务相互等待对方执行的结果，则会死锁：

     ```python
     import time
     def wait_on_b():
         time.sleep(5)
         print(b.result())  # b will never complete because it is waiting on a.
         return 5

     def wait_on_a():
         time.sleep(5)
         print(a.result())  # a will never complete because it is waiting on b.
         return 6

     executor = ThreadPoolExecutor(max_workers=2)
     a = executor.submit(wait_on_b)
     b = executor.submit(wait_on_a)
     ```

   - 如果一个`executor` 任务在等待自己的另一个任务的执行结果，可能会死锁：

     ```python
     def wait_on_future():
         f2 = executor.submit(pow, 5, 2)
         # f2 必须先结束，f1 才能结束。而由于只有一个工作线程，因此是 f1 先执行先结束之后，f2 才能执行。因此死锁。
         print(f2.result())

     executor = ThreadPoolExecutor(max_workers=1)
     f1=executor.submit(wait_on_future)
     ```

### 2.3 ProcessPoolExecutor

1.  `ProcessPoolExecutor` 是 `Executor` 的子类，它使用进程池来执行异步调用

   - 它使用`multiprocessing` 模块，可以有效避开`GIL` 的限制。

2. `API`：

   ```python
   class concurrent.futures.ProcessPoolExecutor(max_workers=None)
   ```

   参数：

   - `max_workers`：进程池的容量（最多`max_workers` 个工作进程）
     - 如果为`None`，则使用`CPU` 的数量
     - 如果小于等于0，则抛出`ValueError` 异常

3. 如果进程池中某个工作子进程异常退出，则抛出`BrokenProcessPool` 异常。

4. `__main__` 模块必须要能够被工作子进程导入，这意味着`ProcessPoolExecutor` 不能工作在交互式环境中。

5. 在工作任务中访问另一个任务的结果，则可能会导致死锁。

### 2.4 Future

1. `Future` 对象封装了异步调用的结果。

   - 它只能由`Executor.submit()` 返回，而不能直接创建

2. API：

   ```python
   class concurrent.futures.Future
   ```

   方法：

   - `.cancel()`：试图取消该异步调用。
     - 如果该异步调用已经开始执行，并且无法被取消，则返回`False`
     - 否则取消异步调用，并且返回`True`
   - `.cancelled()`：如果异步调用被成功取消，则返回`True`
   - `.running()`：如果异步调用正在被执行并且无法被取消，则返回`True`
   - `.done()`：如果异步调用顺利执行结束，或者成功被取消，则返回`True`
   - `.result(timeout=None)`：等待并返回异步调用执行的结果。
     - 如果异步调用未完成，则最多等待`timeout` 秒。如果超时，则抛出`concurrent.futures.TimeoutError` 异常。
     - 如果`timeout=None`，则永不超时。
     - 如果异步调用结束之前被取消，则该方法抛出`.CancelledError` 异常
     - 如果异步调用本身（即`Executor` 初始化函数中的`fn` ）抛出异常，则该方法重新抛出同样的异常
   - `.exception(timeout=None)`：返回异步调用本身（即`Executor` 初始化函数中的`fn`  ）抛出的异常
     - 如果异步调用尚未完成，则最多等待`timeout` 秒。如果超时，则抛出`concurrent.futures.TimeoutError` 异常。
     - 如果`timeout=None`，则永不超时。
     - 如果异步调用结束之前被取消，则该方法抛出`.CancelledError` 异常
     - 如果异步调用结束单身未抛出异常，则该方法返回`None`
   - `.add_done_callback(fn)`： 向`Future` 添加回调函数。
     - `fn` 是回调函数（与`Executor` 初始化函数中的`fn` 不是一个意义）。该回调函数的参数就是`Future` 对象本身。
     - `fn` 回调的时机：异步调用顺利结束时，或者异步调用被成功取消时。
     - 你可以通过使用该方法增加多个回调函数。这些回调函数执行的顺序就是它们添加的顺序。
     - `fn` 回调函数是在调用`.add_done_callback` 的进程中的某个线程中执行（而不是异步调用所在的进程）
     - 如果`fn` 抛出了一个`Exception` 及其子类对象，则该异常会被忽略并记入日志。如果抛出一个`BaseException` 及其子类对象，则行为未定义
     - 如果异步调用已经成功结束，或者顺利被取消，则`fn` 会立即调用。
   - 还有三个方法仅仅用于单元测试。这几个方法只能由`Executor` 的子类实现中被调用：
     - `.set_running_or_notify_cancel()`：如果返回`False` 则表示`Future` 被取消；如果返回`True` 则表示`Future` 未被取消同时处于执行状态
     - `.set_result(result)`：设置`Future` 的结果
     - `.set_exception(exception)`：设置`Future` 的异常

### 2.5 module function

1. `concurrent.futures.wait(fs, timeout=None, return_when=ALL_COMPLETED)`：等待一些`Future` 结束。
   - 参数：
     - `fs` ：一个`Future` 的序列。这些`Future` 可能由不同的`Executor` 创建。
     - `timeout`：控制超时时间。如果超时，则抛出`TimeoutError` 异常。如果为`None` 则永不超时
     - `return_when`：控制该函数的返回时机。可以为下列的值之一：
       - `FIRST_COMPLETED`：只要有任何`Future` 成功结束或者成功被取消，则返回
       - `FIRST_EXCEPTION`：只要有任何`Future` 是以抛出异常而结束，则返回。如果没有任何`Future` 抛出异常，则当所有`Future` 成功结束或者成功被取消，则返回
       - `ALL_COMPLETED`：当所有`Future` 成功结束或者成功被取消，则返回
   - 返回值：返回一个命名元组。为： `(done,not_done)`。其中：
     - `done`：为一个`set`，包含了已经完成的`Future`（顺利结束或者成功被取消）
     - `not_done`：为一个`set`，包含了未完成的`Future`
2. `concurrent.futures.as_completed(fs, timeout=None)`：根据这些`Future` ，返回一个迭代器。
   - 该迭代器迭代的结果就是已经完成（顺利结束或者成功被取消）的`Future` 
   - 如果在迭代器开始迭代的时候，已经有`Future` 完成了，则这些已经完成的`Future` 首先被迭代器取出
   - 如果在迭代的时候(迭代器的`__next__()`方法被调用时)，尚未有`Future` 已经完成，则等待。当等待超时（由`timeout` 给定），则抛出`TimeoutError`。`timeout=None` 则永不超时。
   - 参数：
     - `fs`：一个`Future` 的序列。这些`Future` 可能由不同的`Executor` 创建。
     - `timeout`：控制超时时间。如果超时，则抛出`TimeoutError` 异常。如果为`None` 则永不超时

### 2.6 exception

1. `futures` 模块有三个异常类：
   - `concurrent.futures.CancelledError`： 当`Future` 被取消时被抛出
   - `concurrent.futures.TimeoutError`： 当`Future` 超时的时候被抛出
   - `concurrent.futures.process.BrokenProcessPool`： 当`ProcessPoolExecutor` 中的某个工作进程被异常终止时抛出。它是`RuntimeError` 的子类。



