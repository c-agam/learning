**一、非阻塞IO模型**

![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/IO/Nonblocking%20IO.png)

**二、代码演示**
```
// C端
public class Client throws Exception {
    public static void main(String[] args) {
        SocketChannel socketChannel = SocketChannel.open();
        // 设置为非阻塞
        socketChannel.configureBlocking(false);
        socketChannel.connect(new InetSocketAddress(8080));
        // 等待连接完成
        socketChannel.finishConnect(); 

        // 发送数据
        byte[] bytes = "1234567890".getBytes();
        ByteBuffer byteBuffer = ByteBuffer.wrap(bytes);
        socketChannel.write(byteBuffer);

        // 接收数据
        ByteBuffer responseBuffer = ByteBuffer.allocate(1024);
        socketChannel.read(responseBuffer);
        
        // 关闭操作
        socketChannel.close();
        
    }
}

// S端
public class Server {
    public static void main(String[] args) throws Exception {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);

        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept();
            if (!Objects.isNull(socketChannel)) {
                // 接收数据
                ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                socketChannel.read(requestBuffer);
                
                // 发送数据
                ByteBuffer responseBuffer = ByteBuffer.wrap("1234567890".getBytes());
                socketChannel.write(responseBuffer);
                
                // 关闭操作
                socketChannel.close();
            }
        }
    }
}
```

**三、总结**
```
/**
 * A socket channel in non-blocking mode cannot read any more bytes than are immediately available
 * from the socket's input buffer;
 */
public int read(ByteBuffer dst) throws IOException;

/**
 * A socket channel in non-blocking mode cannot
 * write any more bytes than are free in the socket's output buffer.
 */
public int write(ByteBuffer src) throws IOException;

/**
 * A socket channel is in non-blocking mode then this method will
 * immediately return <tt>null</tt> if there are no pending connections.
 */
public abstract SocketChannel accept() throws IOException;
```
