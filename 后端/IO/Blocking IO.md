**一、阻塞IO模型**

![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/IO/Blocking%20IO.png)

**二、代码演示**

**示例1**
```
// S端
public class Server {
 
    public static void main(String[] args) throws IOException {
 
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
 
        // 监听 8080 端口进来的 TCP 链接
        serverSocketChannel.socket().bind(new InetSocketAddress(8080));
 
        while (true) {
 
            // 这里会阻塞，直到有一个请求的连接进来
            SocketChannel socketChannel = serverSocketChannel.accept();
 
            // 开启一个新的线程来处理这个请求，然后在 while 循环中继续监听 8080 端口
            SocketHandler handler = new SocketHandler(socketChannel);
            new Thread(handler).start();
        }
    }
}

// C端
public class Client {
    public static void main(String[] args) throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress("localhost", 8080));
 
        // 发送请求
        ByteBuffer buffer = ByteBuffer.wrap("1234567890".getBytes());
        socketChannel.write(buffer);
 
        // 读取响应
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        int num;
        if ((num = socketChannel.read(readBuffer)) > 0) {
            readBuffer.flip();
 
            byte[] re = new byte[num];
            readBuffer.get(re);
 
            String result = new String(re, "UTF-8");
            System.out.println("返回值: " + result);
        }
    }
}
```
**示例2**
```
// S端
public class Server {

    public static void main(String[] args) {
        try {
            ServerSocket serverSocket = new ServerSocket(8080);
            while(true) {
                //这里会阻塞，直到有一个请求的连接进来
                Socket socket = serverSocket.accept();
                new Thread(new ServerHandler(socket)).start();
            }
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }
}

// C端
public class Client {

    public static void main(String[] args) {
        Socket socket = new Socket("localhost", 8080);
        OutputStream outputStream = socket.getOutputStream();
        socket.getOutputStream().write("1234567890".getBytes("UTF-8"));
        outputStream.close();
        socket.close();
    }
}
```
**三、总结**
```
/**
 * channel is in blocking mode and there is at least one byte
 * remaining in the buffer then this method will block until at least one
 * byte is read.
 */
 public int read(ByteBuffer dst);

/*
 * Unless otherwise specified, a write operation will return only after
 * writing all of the r requested bytes.  Some types of channels,
 * depending upon their state, may write only some of the bytes or possibly
 * none at all.
 */
 public int write(ByteBuffer src) throws IOException;
 
 /**
 * it will block indefinitely until a new connection is available
 * or an I/O error occurs.
 */
 public abstract SocketChannel accept() throws IOException;
```
