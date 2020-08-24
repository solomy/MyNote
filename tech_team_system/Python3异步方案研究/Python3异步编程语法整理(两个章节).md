## `Python3`异步编程语法整理

#### 现在整理了关于“事件循环”部分的内容（共两个章节）

---

#### Base Event Loop`（基础的事件循环）

- ##### 以下方法都来自于`AbstractEventLoop`(它不是线程安全的)

  - 链接：https://docs.python.org/3.6/library/asyncio-eventloop.html
  - 备注：
    
- 该类下面的方法中，凡是有回调函数的，都需要用`functools.partial`把参数注入到回调函数中
    
- 运行一个事件循环：
  
    - `run_forever`()：让事件循环一直运行，该函数停止运行的标志是：`stop`()被调用
    - `run_until_complete`(*future*)：
    - `is_running`()：返回事件循环的执行状态
    - `stop`()：停止事件循环
    - `is_closed`()：判断事件循环是否已经被关闭了
  - `close`()：关闭事件循环
  
- 回调：
  
    - 需要用到的函数：`functools.partial()`
    - `call_soon(callback, *args)`：该函数执行完之后会马上执行callback回调函数，`*args`是回调函数的参数
  - `call_soon_threadsafe(callback, *args)`：和上面的`call_soon`一样，只不过它是线程安全的
  
- 延迟回调：
  
    - `call_later(delay, callback, *args)`：和上面的相比，就是多了一个延时的参数
    - `call_at(when, callback, *args)`：什么时候调用
  - `time`()：返回当前的时间
  
- `Future`：
  
  - `create_future`()：创建一个`asyncio.Future`对象，并将其附着在循环上
  
- `Task`：
  
    - `create_task(coro)`：创建一个任务，并返回一个Task对象
    - `set_task_factory`(*factory*)：设置一个可供`create_task(coro)`调用的任务工厂
  - `get_task_factory`()：获取到设置的任务工厂
  
- 创建连接：
  
  - `create_connection(protocol_factory, host=None, port=None, *, ssl=None, family=0, proto=0, flags=0, sock=None, local_addr=None, server_hostname=None)`：这是一个协程，它会创建到给定Internet主机和端口的流传输连接，创建成功之后，会返回一个(transport, protocol)元组
  
    （**这里做一个小拓展：函数的关键字参数**：https://www.liaoxuefeng.com/wiki/1016959663602400/1017261630425888）
  
  - `create_datagram_endpoint(protocol_factory, local_addr=None, remote_addr=None, *, family=0, proto=0, flags=0, reuse_address=None, reuse_port=None, allow_broadcast=None, sock=None)`：创建数据报连接，创建成功之后，会返回一个(transport, protocol)元组
  
- 创建监听连接：
  
    - `create_server(protocol_factory, host=None, port=None, *, family=socket.AF_UNSPEC, flags=socket.AI_PASSIVE, sock=None, backlog=100, ssl=None, reuse_address=None, reuse_port=None)`：创建一个TCP连接，并绑定到主机和端口上
    - `create_unix_server(protocol_factory, path=None, *, sock=None, backlog=100, ssl=None)`：针对于`Unix`系统
  - `BaseEventLoop.connect_accepted_socket(protocol_factory, sock, *, ssl=None)`：处理已经接收的连接，当函数执行完成之后，它会返回一个(transport, protocol)元组（**备注**：文章开头写到：`BaseEventLoop should not be subclassed by third-party code; the internal interface is not stable.`，表示它的接口是不稳定的，但是在后面的接口介绍中，又提到了这个函数）
  
- 观看文件描述符（**这部分几乎不支持Windows环境**）：
  
    - `SelectorEventLoop`：在windows环境下，只支持对套接字的处理，其他的都不支持
    - `SelectorEventLoop`：不支持windows环境，官网中，针对该类，主要介绍了四个方法：
      - `add_reader(fd, callback, *args)`：观察文件描述符的“读”可用性，完成后执行回调
      - `remove_reader(fd)`：停止观察文件描述符的“读”可用性
      - `add_writer(fd, callback, *args)`：观察文件描述符的“写”可用性，完成后执行回调
    - `remove_writer(fd)`：停止观察文件描述符的“写”可用性
  
- 低水平的套接字操作：
  
    - `sock_recv(sock, nbytes)`：从套接字中获取数据
    - `sock_sendall(sock, data)`：发送数据到套接字
    - `sock_connect(sock, address)`：根据地址连接到远程的套接字
  - `sock_accept(sock)`：接收一个连接
  
- 解析主机名：
  
    - `AbstractEventLoop.getaddrinfo(host, port, *, family=0, type=0, proto=0, flags=0)`：非阻塞
  - `getnameinfo(sockaddr, flags=0)`：非阻塞
  
- 管道连接：
  
    - windows环境下：用`ProactorEventLoop`类做有关于管道的支持；非Windows环境下，用`SelectorEventLoop`类做管道的支持
    - `connect_read_pipe`(*protocol_factory*, *pipe*)：在事件循环中注册“读”管道
  - `connect_write_pipe`(*protocol_factory*, *pipe*)：在事件循环中注册“写”管道
  
- `Unix`信号
  
    - `add_signal_handler(signum, callback, *args)`：给一个信号添加处理器
  - `remove_signal_handler(sig)`：移除该信号的处理器
  
- 执行器：
  
    - `run_in_executor(executor, func, *args)`：在一个指定的执行器里面运行函数
  - `set_default_executor(executor)`：设置默认执行器
  
- 关于错误处理的一些`API`：
  
    - `set_exception_handler(handler)`：给新的事件循环设置异常处理器
    - `get_exception_handler`()：获取异常处理器
    - `default_exception_handler(context)`：默认异常处理
  - `call_exception_handler`(*context*)：调用当前事件循环的异常处理器
  
- 一些关于事件循环的代码示例（摘自官网）：
  
  - 用延迟回调的方式显示当前时间：
  
      - ```python
        import asyncio
        import datetime
        
        def display_date(end_time, loop):
            print(datetime.datetime.now())
            if (loop.time() + 1.0) < end_time:
                loop.call_later(1, display_date, end_time, loop)
            else:
                loop.stop()
        
        loop = asyncio.get_event_loop()
        
        # Schedule the first call to display_date()
        end_time = loop.time() + 5.0
        loop.call_soon(display_date, end_time, loop)
        
        # Blocking call interrupted by loop.stop()
        loop.run_forever()
        loop.close()
      ```
  
  - 观看文件描述符来监听读取事件：
  
      - ```python
        import asyncio
        try:
            from socket import socketpair
        except ImportError:
            from asyncio.windows_utils import socketpair
        
        # Create a pair of connected file descriptors
        rsock, wsock = socketpair()
        loop = asyncio.get_event_loop()
        
        def reader():
            data = rsock.recv(100)
            print("Received:", data.decode())
            # We are done: unregister the file descriptor
            loop.remove_reader(rsock)
            # Stop the event loop
            loop.stop()
        
        # Register the file descriptor for read event
        loop.add_reader(rsock, reader)
        
        # Simulate the reception of data from the network
        loop.call_soon(wsock.send, 'abc'.encode())
        
        # Run the event loop
        loop.run_forever()
        
        # We are done, close sockets and the event loop
        rsock.close()
        wsock.close()
        loop.close()
      ```
  
  - 设置信号处理器
  
      - ```python
        import asyncio
        import functools
        import os
        import signal
        
        def ask_exit(signame):
            print("got signal %s: exit" % signame)
            loop.stop()
        
        loop = asyncio.get_event_loop()
        for signame in ('SIGINT', 'SIGTERM'):
            loop.add_signal_handler(getattr(signal, signame),
                                    functools.partial(ask_exit, signame))
        
        print("Event loop running forever, press Ctrl+C to interrupt.")
        print("pid %s: send SIGINT or SIGTERM to exit." % os.getpid())
        try:
            loop.run_forever()
        finally:
            loop.close()
        ```



---

#### Event Loops（事件循环 ）

- ##### 以下 方法/类 都来自于`asyncio`模块

  - 几个方便的全局配置函数(位于：`AbstractEventLoopPolicy`中)：

    - `get_event_loop`()：获取事件循环对象，等同于：`get_event_loop_policy().get_event_loop()`
    - `set_event_loop`(*loop*)：设置事件循环对象，等同于：`get_event_loop_policy().set_event_loop(loop)`
    - `new_event_loop`()：创建并返回一个新的事件循环对象， 等同于：`get_event_loop_policy().new_event_loop()`

  - 两个关于事件循环的类：

    - `SelectorEventLoop`（继承于`AbstractEventLoop`）：在windows环境下， 只支持套接字的相关操作；支持Unix环境
    - `ProactorEventLoop`：只支持Windows环境

  - 操作系统的支持和限制，可以直接看官网的介绍：

    https://docs.python.org/3.6/library/asyncio-eventloops.html#platform-support

  - 事件循环策略：

    - 对于大部分的使用者来说，默认配置就足够了，官方还提供了`get_event_loop()`和`set_event_loop()`这两个模块级别的函数去访问默认配置下的事件循环

    - `get_event_loop_policy`()：获取当前事件循环的策略

    - `set_event_loop_policy`(*policy*)：设置当前事件循环的策略

    - 示例：

      ```python
      class MyEventLoopPolicy(asyncio.DefaultEventLoopPolicy):
      
          def get_event_loop(self):
              """Get the event loop.
      
              This may be None or an instance of EventLoop.
              """
              loop = super().get_event_loop()
              # Do something with loop ...
              return loop
      
      asyncio.set_event_loop_policy(MyEventLoopPolicy())
      ```

---

##### 参考链接：https://docs.python.org/3.6/library/asyncio.html

