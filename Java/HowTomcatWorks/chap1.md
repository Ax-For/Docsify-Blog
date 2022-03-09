# 第一章：一个简单的Web服务器

### HTTP 超文本传输协议

#### 1.HTTP请求
一个HTTP请求包括三个组成部分：

- 【方法】【统一资源标识符(URI)】【协议/版本】
- 请求首部
- 内容实体

![](pics/Snipaste_2022-03-09_16-50-25.png)

下面是一个`HTTP`请求的例子
```html
POST /examples/default.jsp HTTP/1.1
Accept: text/plain; text/html
Accept-Language: en-gb
Connection: Keep-Alive
Host: localhost
User-Agent: Mozilla/4.0 (compatible; MSIE 4.01; Windows 98)
Content-Length: 33
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate

lastName=Franks&firstName=Michael
```
请求首部包含了关于客户端环境和请求的主题内容的有用信息。头部之间通过一个`回车换行符(CRLF)`进行分割。

请求首部与内容实体之间通过一个`回车换行符(CRLF)`分割。

#### 2.HTTP响应
类似于HTTP请求，HTTP响应也包括三个组成部分：

- 【协议/版本】【状态码】【状态码的原因短语】
- 响应首部
- 响应主体

![](pics/Snipaste_2022-03-09_17-00-57.png)

下面是一个`HTTP`响应的例子
```html
HTTP/1.1 200 OK
Server: Microsoft-IIS/4.0
Date: Mon, 5 Jan 2004 13:13:33 GMT
Content-Type: text/html
Last-Modified: Mon, 5 Jan 2004 13:13:12 GMT
Content-Length: 112

<html>
<head>
<title>HTTP Response Example</title>
</head>
<body>
Welcome to Brainy Software
</body>
</html>

```

响应头部和请求头部类似,也包括了很多有关响应的有用信息，各头部之间通过一个`回车换行符(CRLF)`进行分割。

响应的主体内容可以是响应本身的HTML（也可以是其他内容），头部和主体之间通过一个`回车换行符(CRLF)`分割。

### socket连接

> 套接字（socket）是网络连接的一个端点。套接字使得一个应用可以从网络读取和写入数据。放在两个不同计算机上的两个应用可以通过套接字连接发送和接收字节流。

套接字 = IP地址 + 端口

对应的，在Java中，套接字主要通过两个类实现。客户端`Socket`和服务器端`ServerSocket`

#### Socket   

对于一个客户端应用，当需要建立`socket`连接时，需要用到
`java.net.Socket`类

构造方法：
```java
public Socket (java.lang.String host, int port)
```
`host`：远程机器名称或IP地址 

`port`：远程应用的端口号

一旦创建了一个Socket的实例，就可以通过这个实例来发送和接收字节流。

- 要发送`字节流`，需要调用`Socket`的`getOutputStream()`方法来获取一个`java.io.OutputStream`对象。

- 要发送`文本`到一个远程应用,则需要从获得的`OutputStream`对象中构造一个`java.io.PrintWritter`。

- 要从连接的另一端接收字节流，可以调用`Socket`的`getInputStream()`方法来返回一个`java.io.InputStream`对象。


以下的代码片段创建套接字，可以和本地的`HTTP`服务器进行通讯，发送请求并获取响应，最后将响应在控制台上打印出来。

```java
try(Socket socket = new Socket("127.0.0.1", 9094);
    OutputStream os = socket.getOutputStream();
    PrintWriter writer = new PrintWriter(os, true);
    // 针对输出文本流的读取
    InputStream is = socket.getInputStream();
    BufferedReader reader = new BufferedReader(new InputStreamReader(is))){
        writer.print("GET /index.html HTTP/1.1\r\n");
        writer.print("Host: 127.0.0.1:9094\r\n");
        writer.print("Connection: Close\r\n");
        writer.print("\r\n");
        // 强制刷新，清空缓存区
        writer.flush();
        StringBuilder sb = new StringBuilder(8096);
        String line;
        // 阻塞式读取
        while((line = reader.readLine()) != null){
            sb.append(line).append('\n');
        }
        System.out.println(sb.toString());
    } catch (IOException e) {
        e.printStackTrace();
}
```
客户端连接远程服务器`127.0.0.1:9094`，发送`get`请求，并将服务器的`response`以字符流的形式读取并输出在控制台上。

有了客户端请求的配置，接下来还需要配置服务器端的套接字

#### ServerSocket

对于一个服务器应用，如HTTP服务器或者FTP服务器，建立套接字的做法与客户端不同，因为服务器必须随时待命，因为不知道客户端应用什么时候会尝试去连接它。为了让应用能随时待命，需要使用服务器套接字类`java.net.ServerSocket`类。

`ServerSocket`与`Socket`不同，主要功能是等待来自客户端的连接请求。一旦服务器套接字获得一个连接请求，它会创建一个`Socket`实例来与客户端进行通信。

要创建一个服务器套接字。需要使用`ServerSocket`类提供的构造方法，需要指定IP地址和监听端口号，通常来说，IP地址就是服务器套接字运行的本机地址`127.0.0.1`，也就是说，套接字需要监听本地机器；除此之外，服务器套接字的另一个重要的属性是`backlog`，这是服务器套接字开始拒绝传入的请求之前，传入的连接请求的最大队列长度。

其中一个`ServerSocket`类的构造方法如下所示：
```java
public ServerSocket(int port, int backLog, InetAddress bindingAddress);
```

`socket`客户端和服务器的连接顺序如下图所示：

![](pics/2184951-feff3fcc300fcfcd.png)

期间经过三次握手：

`1`:`client -> server (SYN)`

在这个阶段，`client`发送`SYN`到`server`，将自身状态改为`SYN_SEND`，如果`server`收到请求，则将状态修改为`SYN_RCVD`，并把该请求放到`syns queue`队列中

`2`:`server -> client (SYN + ACK)`

在这个阶段，`server`回复`SYN + ACK`给`client`，如果`client`收到请求，则将自身状态修改为`ESTABLISHED`

`3`:`client -> server (ACK)`

在这个阶段，`client`发送`ACK`给`server`，如果`server`收到请求，则把自身状态修改为`ESTABLISHED`，并把该请求从`syns queue`中放到`accept queue`。

在这个过程中，系统内核维护了两个队列：`syns queue`和`accept queue`，传入的参数`backlog`限制的就是`syns queue`队列的大小。


一旦获取了一个`ServerSocket`实例，可以在绑定监听端口后等待传入的连接请求。可以通过调用`ServerSocket`实例的`accept()`方法获取`socket`实例，可以从中获取请求信息，以及发送响应信息。


整个`web`服务器由三个类组成：
 
- HttpServer
- Request
- Response

接下来分别介绍三个类的实现

首先是`Request`类

```java
public class Request {

    private static final int BUFF_SIZE = 2048;
    private final InputStream is;
    private String uri;

    public Request(InputStream is) {
        this.is = is;
    }

    // 对输入进行解析，获取输入流
    public void parse() {
        System.out.println("开始Request解析");
        StringBuilder sb = new StringBuilder(BUFF_SIZE);
        byte[] buffer = new byte[BUFF_SIZE];
        int i;
        try {
            System.out.println("开始读取输入流");
            if ((i = is.read(buffer, 0, BUFF_SIZE)) != -1) {
                for (int j = 0; j < i; j++) {
                    sb.append((char)buffer[j]);
                }
            }
            System.out.println("输入流读取完成");
        } catch (IOException e) {
            e.printStackTrace();
        }
        System.out.println("请求信息解析完成");
        System.out.println(sb);
        uri = parseUri(sb.toString());
    }

    // 获取uri
    public String getUri() {
        return uri;
    }

    // 从请求中解析出uri
    private String parseUri(String requestString) {
        if (requestString == null) {
            return null;
        }
        int idx1 = requestString.indexOf(' ');
        if (idx1 != -1) {
            int idx2 = requestString.indexOf(' ', idx1 + 1);
            if (idx2 > idx1) {
                return requestString.substring(idx1 + 1, idx2);
            }
        }
        return null;
    }
}
```