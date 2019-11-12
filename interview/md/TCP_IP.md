## Socket、ServerSocket的区别
https://www.cnblogs.com/douzi520/p/9303179.html

##### 不妨先考虑一个问题：在上一节的实例中，我们运行 echo 服务后，在客户端连接成功时，一个有多少个 socket 存在？

> 答案是3个 socket：客户端一个，服务端有两个。

在 Java 的 SDK 中，socket 的共有两个接口：用于监听客户连接的 ServerSocket 和用于通信的 Socket。

>注意：我只说 ServerSocket 是用于监听客户连接，而没有说它也可以用来通信。下面我们来详细了解一下他们的区别。
以下描述使用的是 UNIX/Linux 系统的 API。

首先，我们创建 ServerSocket 后，内核会创建一个 socket。这个 socket 既可以拿来监听客户连接，也可以连接远端的服务。由于 ServerSocket 是用来监听客户连接的，紧接着它就会对内核创建的这个 socket 调用 listen 函数。这样一来，这个 socket 就成了所谓的 listening socket，它开始监听客户的连接。

接下来，我们的客户端创建一个 Socket，同样的，内核也创建一个 socket 实例。内核创建的这个 socket 跟 ServerSocket 一开始创建的那个没有什么区别。不同的是，接下来 Socket 会对它执行 connect，发起对服务端的连接。前面我们说过，socket API 其实是 TCP 层的封装，所以 connect 后，内核会发送一个 SYN 给服务端。

现在，我们切换角色到服务端。服务端的主机在收到这个 SYN 后，会创建一个新的 socket，这个新创建的 socket 跟客户端继续执行三次握手过程。

三次握手完成后，我们执行的 serverSocket.accept() 会返回一个 Socket 实例，这个 socket 就是上一步内核自动帮我们创建的。

所以说：在一个客户端连接的情况下，其实有 3 个 socket。

关于内核自动创建的这个 socket，还有一个很有意思的地方。它的端口号跟 ServerSocket 是一毛一样的。咦！！不是说，一个端口只能绑定一个 socket 吗？其实这个说法并不够准确。

前面我说的TCP 通过端口号来区分数据属于哪个进程的说法，在 socket 的实现里需要改一改。Socket 并不仅仅使用端口号来区别不同的 socket 实例，而是使用 <peer addr:peer port, local addr:local port> 这个四元组。

在上面的例子中，我们的 ServerSocket 长这样：<* : *, *:9877>。意思是，可以接受任何的客户端，和本地任何 IP。

accept 返回的 Socket 则是这样：<127.0.0.1:xxxx, 127.0.0.1:9877>。其中，xxxx 是客户端的端口号。

如果数据是发送给一个已连接的 socket，内核会找到一个完全匹配的实例，所以数据准确发送给了对端。

如果是客户端要发起连接，这时候只有 <* : *, *:9877> 会匹配成功，所以 SYN 也准确发送给了监听套接字。




### TCP/IP详解 卷1：协议 第一版
http://www.52im.net/topic-tcpipvol1.html?mobile=no