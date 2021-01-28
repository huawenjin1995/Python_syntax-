# 外部程序

## 一、外部程序

1. 外部程序由 `subprocess` 模块提供

###1.1 run 

1.  执行外部程序最简单的，也是推荐的方式是使用 `subprocess.run()` 函数。

2. 函数声明：

   ```python
   subprocess.run(args, *, stdin=None, input=None, stdout=None, stderr=None, shell=False, cwd=None, timeout=None, check=False, encoding=None, errors=None)
   ```

   该函数执行由`args` 指定的命令（创建一个子进程），等待命令执行结束，并且返回一个`CompletedProcess` 实例。

   参数：

   - `args`：待执行的命令。可以为：

     - 一个字符串：表示待执行的命令。如 `'ls -l ./'`
     - 一个字符串列表：表示待执行的命令及其参数。如 `['ls','-l','./']`。

     > 通常建议使用字符串列表，因为它自动处理参数。比如参数是个文件名，而文件名带有空格符的情况。

   - `stdin/stdout/stderr`：设子进程的标准输入、标准输出、标准错误。可以为：

     - `subprocess.PIPE`：为子进程创建一个新的管道。
     - `subprocess.DEVNULL`：使用特殊的文件`os.devnull` 
     - 一个现有的文件描述符（一个正整数）：使用指定的文件
     - 一个现有的文件对象：使用指定的文件
     - `None`：子进程从父进程继承这三个配置。

   - `shell`  ：如果为`True`，则命令通过`shell`执行。其优点是：你可以在命令中嵌入一些 `shell` 的特性，如输出重定向，特殊命令等。

   - `timeout`   ：该参数将传递给`Popen.communicate()`。如果超时，则子进程被杀死。父进程捕获到子进程被杀死之后抛出`TimeoutExpired` 异常。

     > 两个条件：父进程等待子进程结束、子进程超时。只要有一个不满足，则父进程不会抛出 `TimeoutExpired` 异常。

   - `input`： 该参数将传递给`Popen.communicate()`，因此最终会被传递给子进程的`stdin`。

     - 如果使用该参数，则`stdin`参数被忽略。同时子进程自动创建`stdin=PIPE`

     - 该参数要么是一个字节序列，或者是个字符串。它将作为管道的输入数据。

       > 当时字符串时，必须指定  encoding、errors 之一，或者 universal_newlines=True

   - `check` ：如果为 `True`，则会校验子进程的退出码：如果子进程的非零的退出码退出，则抛出`CalledProcessError` 实例。

   - 上述参数只是一些常用的参数，还有更多的参数见官方文档。完整的参数与`Popen` 构造函数的参数几乎相同

3. 当  `encoding`、`errors` 设置，或者  `universal_newlines=True` 时，`stdin,stdout,stderr` 将以文本模式（`encoding` 指定的编码） 打开：

   - 对于`stdin` 中的换行符` '\n'` 将被转换为默认的换行符 ` 'os.linesep'`。对于 `stdout,stderr` 中的换行符，统一被转换为` '\n' `。

   否则， `stdin,stdout,stderr `以二进制模式打开，此时不执行任何换行符的转换

4. `subprocess.CompletedProcess` 实例：代表了 `run()` 函数的返回值。其属性有：

   - `args`：子进程的参数，是一个列表或者字符串。是`run` 的`args` 参数
   - `returncode`：子进程的退出状态。
     - 通常，如果为 0 表示子进程成功执行。
     - `-N` 表示子进程被信号 `N` （N 为任意整数，如 10） 终止 （`POSIX` ）
   - `stdout`：捕获到的子进程的`stdout` 输出。
     - 可以为一个字节序列、字符串。取决于`run` 的`encoding,errors` 参数。
     - 也可以是`None`，如果没有捕获`stdout` 
   - `stderr`：捕获到的子进程的`stderr` 输出。
     - 可以为一个字节序列、字符串。取决于`run` 的`encoding,errors` 参数。
     - 也可以是`None`，如果没有捕获`stderr` 
     - 如果子进程的参数为 `stderr=subprocess.STDOUT`，则子进程的`stderr`  也输出到`stdout` 中。那么这里的`stderr` 也是 `None`
   - `check_returncode()` 方法：如果`returncode` 非0，则抛出 `CalledProcessError` 

5. `subprocess.TimeoutExpired`  实例：表示子进程超时异常。其属性有：

   - `cmd`：一个字符串，表示产生子进程的命令
   - `timeout`：表示设置的超时时间（单位秒）
   - `output`：子进程的标准输出（由`run()`或者 `check_output()`  函数捕获）。如果未捕获，则返回`None`
   - `stdout` ： `output` 的别名
   - `stderr`： 子进程的标准出错（由 `run()` 函数捕获）。如果未捕获，则返回`None`

6. `sunprocess.CalledProcessError` 实例：当子进程返回一个非零的退出码，且父进程执行了 `check_all()/check_output()` 时产生。其属性有：

   - `returncode` ：子进程的退出码。
     - 如果子进程由于某个信号退出，则退出码就是该信号的编号的负值
   - `cmd`：一个字符串，表示产生子进程的命令
   - `output/stdout/stderr` ：同 `TimeoutExpired`

### 1.2 Popen

1. `run` 的底层是由 `Popen` 实现的。`Popen` 能实现更精细的控制，它是多进程的基石。

2. `Popen` 的初始化函数：

   ```python
   subprocess.Popen(args, bufsize=-1, executable=None, stdin=None, stdout=None, stderr=None, preexec_fn=None, close_fds=True, shell=False, cwd=None, env=None, universal_newlines=False, startupinfo=None, creationflags=0, restore_signals=True, start_new_session=False, pass_fds=(), *, encoding=None, errors=None)
   ```

   它创建子进程来执行指定命令。

   -  在`POSIX` 系统下，它调用`os.execvp()` 函数
   -  在`Windows` 系统下，它调用`CreateProcess()` 函数

3. `Popen` 的`args` 参数：一个字符串或者一个字符串序列，指定待执行的命令

   - 如果为字符串序列，则第一个元素默认为程序名
   - 如果为字符串，则该字符串指定了待执行的命令。

   该参数的解释取决于不同的平台：

   - 在`POSIX` 平台，如果`args` 为一个字符串：

     - 当`shell=False` 时，该字符串解释为待执行的程序的程序名，或者程序路径。此时无法传入命令参数
     - 当`shell=True` 时，表示在`shell` 中执行命令（默认为`/bin/sh`）。此时`args` 字符串就好像你在`shell` 中输入的命令。如果命令的参数带有空白，则必须转义空白符

   - 在`POSIX` 平台，如果`args` 为一个字符串序列：

     - 当`shell=False` 时：第一个元素默认为程序名，后面的元素为命令参数
     - 当`shell=True` 时：第一个元素默认为程序名，后面的元素为`shell` 本身的参数。即等价于：`Popen(['/bin/sh', '-c', args[0], args[1], ...])` 

   - 在`Windows` 平台：

     - 如果`args` 为字符串序列，则该序列被自动转换为一个字符串。因为`CreateProcess()`函数的参数是一个字符串。
     - 如果`args` 为字符串，则它直接传递给 `CreateProcess()`函数
     - `shell=True`  的讨论也类似`POSIX` 。但是`windows` 中，你用到该参数的唯一场景是：你需要执行`shell` 中内建的命令，如`dir,copy` 

   - `shell=True` 的优点是：你可以在命令中嵌入一些 `shell` 的特性，如输出重定向，特殊命令等。

4. `Popen` 的`bufsize` 参数：子进程的`stdin,stdout,stderr` IO缓冲模式：

   - 0：表示不缓冲。任意读、写操作都是一个系统调用，并且很快返回。
   - 1：表示行缓冲。只在文本模式，且`universal_newline=True` 时起作用
   - 任意正整数：表示缓冲区大小
   - 任意负整数：表示默认缓冲区大小（`io.DEFAULT_BUFFER_SIZE` ）

5. `Popen` 的其他参数：

   - `executable` 很少使用
   - `stdin,stdout,stderr` 参数：见`run()` 的讨论。
   - `preexec_fn` 参数：仅在`POSIX` 平台有效。它是一个可调用对象，在子进程中调用。调用时机为：子进程的命令执行之前。
     - 注意：该功能是线程不安全的，可能导致子进程死锁。尽量少用
     - 如果想为子进程建立环境，则尽量使用`env` 参数
   - `close_fds` 参数：仅在`POSIX` 平台有效。如果为`True`，则子进程的命令执行之前关闭子进程的描述符0,1,2。
   - `pass_fds` 参数：仅在`POSIX` 平台有效。它是个列表，指定在父子进程中仍然保持打开的文件描述符
   - `cwd` 参数：如果不是`None`，则为子进程设置工作目录。该参数的值可以为一个字符串，或者一个`path-like` 对象。
   - ` restore_signals` 参数：仅在`POSIX` 平台有效。`Python` 忽略的信号（行为是`SIG_IGN`），在子进程中都将不再忽略（行为是 `SIG_DFL` ）
     - 目前`python` 忽略的信号有：`SIGPIPE,SIGXFX,SIGXFSZ` 
   - `start_new_session` 参数：仅在`POSIX` 平台有效。如果为`True`，则子进程中，程序执行之前调用`setsid()` 系统调用来创建新的会话。
   - `env` 参数：如果非`None`，则必须是个映射，为子进程定义环境变量。它将替代子进程默认从父进程继承的环境变量。
   - `encoding,errors,universal_newlines` 参数：参见`run` 函数的讨论
   - `startupinfo` 参数：仅在`windows` 下有效。是一个`STARTUPINFO`对象，传递给底层的`CreateProcess`函数

6. `Popen` 对象支持上下文管理器。在退出时，标准输入、标准输出、标准错误文件描述符被关闭，子进程被`wait`：

   ```python
   with Popen(['ls' ,'-la'],stdout=PIPE) as proc:
   	log.write(proc.stdout.read())
   ```

7. `Popen`  的方法：
  - `.poll()` ：检测子进程是否结束。设置并返回`returncode` 属性
  - `.wait(timeout=None)`：等待子进程结束。设置并返回`returncode` 属性。
    - 注意：当使用了 `stdout=PIPE` 或者`stderr=PIPE` 时，如果管道是满的，则子进程可能阻塞在写管道。此时若父进程等待子进程，则会死锁。
    - 这个函数使用一个`busy loop` 来实现。推荐使用`asyncio` 模块来实现异步等待。
  - `.communicate(input=None,timeout=None)`：与子进程交互。通过`input` 向子进程传入数据。从`stdout,stderr` 读取数据。
    - 它返回一个元组 `(stdout_data,stderr_data)` 。如果是文本模式，则它们都是字符串，否则它们都是字节串
    - 该方法会等待子进程结束。
    - 如果超时，并捕获了`TimeoutExpired` 异常并重新调用`.communicate(..)`，则返回的`(stdout_data,stderr_data)` 依然是完整的，并不会丢失。
    - 如果超时，则子进程并不会被杀死。所以当交互成功之后，需要手动执行子进程的清理工作。
  - `.send_signal(signal)`：向子进程发送信号。
  - `.terminate()`：停止子进程。对于`POSIX` 平台，等效于向子进程发送`SIGTERM` 信号；在`windows` 平台，等效于调用`TerminiateProcess()` 函数
  - `.kill()`：杀死子进程。对于`POSIX` 平台，等效于向子进程发送`SIGKILL` 信号；在`windows` 平台，等效于调用`TerminiateProcess()` 函数

8. `Popen` 属性：

   - `.args`：传入的`.args` 参数
   - `.stdin`：如果`stdin`参数为`PIPE`，则它是一个可写的流对象；否则是`None`
   - `.stdout`：如果`stdout`参数为`PIPE`，则它是一个可读的流对象；否则是`None`
   - `.sterr`：如果`sterr`参数为`PIPE`，则它是一个可读的流对象；否则是`None`
   - `.pid`：子进程的进程ID
   - `.returncode` ：子进程的退出码。由`.poll()/.wait()/.communicate()` 方法设置。
     - 如果该值为`None`，则说明子进程尚未结束。
     - 在`POSIX` 平台，如果为负数，则表明子进程由对应的信号停止。 

### 1.3 异常

1. 异常发生在子进程中，在子进程的程序执行之前。同时，异常也会在父进程中重新被抛出
   -  异常对象拥有一个额外的属性，称作`child_traceback` ，它是个字符串，包含了子进程的异常信息
2. 最常见的异常为`OSError`。当执行一个不存在的程序时，会抛出该异常
3. 当`Popen`  提供了无效的参数时，`ValueError` 被抛出
4. 如果子进程的退出码非零，则调用`check_call()/check_output()` 会抛出`CalledProcessError` 异常
5. 大多数函数和方法（如`call(),Popen.commuicate()` ）接受一个`timeout`参数，表示超时时间。如果子进程超时，则抛出`TimeoutExpired` 异常。

## 二、旧的接口

1. 在 `python 3.5` 之前提供了下面三个接口。`python3.5` 之后你可以用`subprocess.run()` 函数来代替：
   - `subprocess.call(args, *, stdin=None, stdout=None, stderr=None, shell=False, cwd=None, timeout=None)` ：执行命令，等待命令结束，并返回退出码
     - 等价于 ` run(...).returncode`
   - `subprocess.check_call(args, *, stdin=None, stdout=None, stderr=None, shell=False, cwd=None, timeout=None)`：执行命令，等待命令结束。如果退出码非0，则抛出 `CalledProcessError` 异常（该异常持有退出码）
     - 等价于 `run(..., check=True)`
   - `subprocess.check_output(args, *, stdin=None, stderr=None, shell=False, cwd=None, encoding=None, errors=None, universal_newlines=False, timeout=None)`： 执行命令，等待命令结束并给出`output` 。如果退出码非0，则抛出 `CalledProcessError` 异常（该异常持有退出码）
     - 等价于 `run(..., check=True, stdout=PIPE).stdout`

