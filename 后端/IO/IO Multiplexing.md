**一、IO多路复用模型**

![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/IO/IO%20Multiplexing.png)

**二、IO多路复用简介**

上图中，应用程序会调用select或epoll，其实这个里面不仅仅如此，只是为了作图方便，列举了这两种。下面我们来具体了解下。

* **select：**
起源于上世纪 80 年代，它最多支持注册 FD_SETSIZE(1024) 个 socket。截止目前windows系统非阻塞依旧采用的是这种方式。
* **poll：**
1997 年，出现了 poll 作为 select 的替代者，最大的区别就是，poll 不再限制 socket 数量。

* **epoll：**
2002 年随 Linux 内核 2.5.44 发布，epoll 能直接返回具体的准备好的通道，时间复杂度 O(1)。
* **kqueue：**
2000 年 FreeBSD 出现了 Kqueue

**select 和 poll** 都有一个共同的问题，那就是它们都只会告诉你有几个通道准备好了，但是不会告诉你具体是哪几个通道。所以，一旦知道有通道准备好以后，自己还是需要进行一次扫描，显然这个不太好，通道少的时候还行，一旦通道的数量是几十万个以上的时候，扫描一次的时间都很可观了，时间复杂度 O(n)。