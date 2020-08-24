#WEBSOCKET

-----
### WebSocket是什么?
WebSocket 是 HTML5 开始提供的⼀种在单个 TCP 连接上进⾏全双⼯通讯的协议
### 为什么需要?
HTTP 协议是⼀种⽆状态的、⽆连接的、单向的应⽤层协议。它采⽤了请
求/响应模型。通信请求只能由客户端发起，服务端对请求做出应答处理
弊端: HTTP 协议⽆法实现服务器主动向客户端发起消息。
传统模式下， Web 应⽤程序通过频繁的 ajax 请求实现⻓轮询( 轮询是在 特定的时间间隔(如每1秒),
由浏览器对服务器发出 HTTP 请求，然后由 服务器返回最新的数据给客户端的浏览器)
缺点:轮询的效率低，⾮常浪费带宽等资源(浏览器需要不断的向服务器
发出请求)

### 如何⼯作?
Web 浏览器和服务器都必须实现 WebSockets 协议来建⽴和维护连 接，由于 WebSockets 连接⻓
期存在，与典型的 HTTP 连接不同，对 服务器有重要的影响(任何 WebSockets 服务器都需要实现
为异步服 务器，基于多线程或多进程的服务器⽆法适⽤于 WebSockets，因为 它旨在打开连接，尽
可能快地处理请求，然后关闭连接)
在 WebSocket 协议中, 浏览器和服务器只需要做⼀个握⼿的动作，然后，浏览器和服务器之间就形
成了⼀条快速通道。两者之间就直接可以数据互相传送。


### 如何使⽤?
客户端 API (javascript)
1、创建 websocket 对象
var ws = new WebSocket(url, [protocol] );
2、属性
ws.readyState 表示连接状态
可选值:
0: 表示连接尚未建⽴。
1: 表示连接已建⽴，可以进⾏通信。
2: 表示连接正在进⾏关闭。
3: 表示连接已经关闭或者连接不能打开。
ws.bufferedAmount 表示已被 send() ⽅法放⼊正在队列中等待传输，但是还没有发 出的 UTF-8 ⽂
本字节数
3、事件
open ws.onopen 建⽴连接时触发
message ws.onmessage 客户端接收服务端数据时触发
error ws.onerror 通信发⽣错误时触发
close ws.onclose 连接关闭时触发
4、⽅法
send ws.send() 使⽤连接发送数据
close ws.close() 关闭连接


[django-websocket-redis](https://github.com/jrief/django-websocket-redis)

采用redis作为消息队列

优点：

- It is simpler to implement.
- The asynchronous I/O loop handling websockets can run
- inside Django with  `./manage.py runserver` , giving full debugging control.
- as a stand alone HTTP server, using uWSGI.
- using NGiNX or Apache (&gt;= 2.4) as proxy in two decoupled loops, one for WSGI and one for websocket HTTP in front of two separate uWSGI workers.
- The whole Django API is available in this loop, provided that no blocking calls are made. Therefore the websocket code can access the Django configuration, the user and the session cache, etc.


websocket. redis:

Of course, here it also is prohibitive to create a new thread for each open websocket connection. Therefore that part of the code runs in one single thread/process for all open connections in a cooperative concurrency mode using the excellent [gevent](http://www.gevent.org/) and [greenlet](http://greenlet.readthedocs.org/) libraries.

### 实例演示



