
**一、前言**

Accept-Encoding：HTTP请求头，标识客户端能够理解的内容编码方式，通常是一种压缩算法，如：gizp，deflate，compress，identity... 客户端向服务端发送该请求头，示意服务端如果需要编码只能用这些编码方式，当然也有可以不用。无论用还是不用，服务端需要在响应头中加入Content-Encoding，其中identity标识未使用编码。

**二、实践**

**客户端**
```
httpConn.setRequestProperty("Accept-Encoding", "gzip,deflate")；
httpConn.setRequestProperty("Content-Type", "application/json;charset=UTF-8")；
```

**服务端**
```
response.setHeader("Pragma", "no-cache")；
response.setContentType("text/plain;charset=UTF-8")；
```

**三、现象**

**编程方式：**
- 如果一次响应数据达到一定的值，客户端获得的数据会出现乱码现象
- 如果一次响应的数据未达到一定的值，客户端获得的数据不会出现乱码现象

**工具模拟：**
- 用jmeter工具模拟客户端请求时，无论响应数据量大与小都不会出现乱码问题

**四、解决**
* 服务端使用客户端提议的一种压缩方式
* 去掉客户端Accept-Encoding请求头
* 服务端使用Content-Encoding作出合适的响应
