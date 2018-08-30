---
layout:     post
title:      "Java Socket 编程"
subtitle:   "Java socket"
date:       2018-06-18 23:30:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Socket
    - 计算机网络
---

> 一切皆Socket！

## Socket 的历史

套接字API起源于1983年发行的4.2BSD（Berkeley Software Distribution 也称为伯克利Unix），然而由于AT&T的专利保护着UNIX，所以只有在1989年伯克利大学才能自由地发布自己的操作系统和网络库。Berkeley套接字应用程序接口形成了网络套接字标准的精髓，被用于Unix域套接字。

Linux这种流行的可免费获取的Unix实现并不适合归属院子Berkeley的系列，因为它的网络支持和套接字API都是从头开发的。90年代初，Microsoft联合了几家其他公司共同制定了一套Windows下的网络编程接口-Windows Sockets规范。目前Windows下的Internet软件都是基于WinSock开发的。

```
在组网领域的首次使用是在1970年2月12日发布的文献IETF RFC33中发现的，撰写者为Stephen Carr、Steve Crocker和Vint Cerf。根据美国计算机历史博物馆的记载，Croker写道：“命名空间的元素都可称为套接字接口。一个套接字接口构成一个连接的一端，而一个连接可完全由一对套接字接口规定。”计算机历史博物馆补充道：“这比BSD的套接字接口定义早了大约12年。”
```

socket起源于Unix，而Unix/Linux基本哲学之一就是“一切皆文件”，都可以用“打开open –> 读写write/read –> 关闭close”模式来操作。Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）。Socket是进程间通信的一种方式，既可以用于同一个主机的两个进程，也可以用于不同主机的两个进程。

网络Socket的基本抽象：

![](http://new.51cto.com/files/uploadimg/20090215/153421796.jpg)

Socket的实现有：

- Unix Socket
- Linux Socket
- WinSock
- Java Socket
- Python Socket

本文主要专注Java Socket的实现，当然底层还是会通过JVM调用系统的API。

## Java Socket基础

#### 套接字地址

**NetworkInterface**：NetworkInterface类提供了访问主机所有接口的信息的功能。(IP地址实际上是分配给了主机与网络之间的连接，而不是主机本身)

**InetAddress**：网络接口，代表了一个网络目标地址，包括主机名和数字类型的地址信息。该类有两个子类，Inet4Address和Inet6Address，分别对应了目前IP地址的两个版本。InetAddress实例是不可变的，一旦创建，每个实例就始终指向同一个地址。

**SocketAddress**：抽象类，代表了套接字地址的一般型式，它的子类InetSocketAddress是针对TCP/IP套接字的特殊型式，封装了一个InetAddress和一个端口号。InetSocketAddress类为主机地址和端口号提供了一个不可变的组合。只接收端口号作为参数的构造函数将使用特殊的"任何"地址来创建实例，这点对于服务器端非常有用。接收字符串主机名的构造函数会尝试将其解析成相应的IP地址。

获取所有的网络接口信息：

```java
Enumeration<NetworkInterface> interfaceList = NetworkInterface.getNetworkInterfaces();
while (interfaceList.hasMoreElements()) {
                    NetworkInterface iface = interfaceList.nextElement();
                    System.out.println("Interface " + iface.getName() + ":");
                    Enumeration<InetAddress> addrList = iface.getInetAddresses();
                    while (addrList.hasMoreElements()) {
                        InetAddress address = addrList.nextElement();
                        System.out.print("Address "
                        + ((address instanceof Inet4Address ? "V4" : (address instanceof Inet6Address ? "V6" : "?"))));
                        System.out.println(": " + address.getHostAddress());
                    }
}
```

其他常用的方法：

```java

//创建和访问
static InetAddress[] getAllByName(String host);
static InetAddress getByName(String host);
static InetAddress getLocalHost();
byte[] getAddress();
String getHostAddress(); //返回数字地址
String getHostName(); // 如果通过主机名创建则直接返回，否则要域名解析
String getCanonicalHostName(); // 总是域名解析
boolean isReachable(); // Ping 判断是否可以访问
```

#### TCP 套接字

**Socket**和**ServerSocket**：Java为TCP协议提供了两个类：Socket类和ServerSocket类。一个Socket实例代表了TCP连接的一端。一个TCP连接（TCP connection）是一条抽象的双向信道，两端分别由IP地址和端口号确定。在开始通信之前，要建立一个TCP连接，这需要先由客户端TCP向服务器端TCP发送连接请求。ServerSocket实例则监听TCP连接请求，并为每个请求创建新的Socket实例。也就是说，服务器端要同时处理ServerSocket实例和Socket实例，而客户端只需要使用Socket实例。

Socket是在TCP连接的基础上实现的，但是在Java中，可以改变Socket的底层连接。本文主要是关于TCP/IP连接的。

简单的TCP客户端操作：

```java
public class TCPEchoClient {
    public static void main(String[] args) throws IOException {
        String server = "localhost";
        int port = 10000;
        byte[] data = "I'm client".getBytes();
        Socket socket = new Socket(server, port);
        System.out.println("Connected to server...sending echo string");
        InputStream in = socket.getInputStream();
        OutputStream out = socket.getOutputStream();
        out.write(data);
        //Receive
        int totalBytesRcvd = 0;
        int bytesRcvd;
        while (totalBytesRcvd < data.length) {
            if ((bytesRcvd = in.read(data, totalBytesRcvd,
                    data.length - totalBytesRcvd)) == -1) {
                throw new SocketException("Connection closed prematurely");
            }
            totalBytesRcvd += bytesRcvd;
        }  // data array is full
        System.out.println("Received: " + new String(data));
        socket.close();  // Close the socket and its streams
    }
}
```

简单的TCP服务端操作：

```java
public class TCPEchoServer {
    private static final int BUFSIZE = 32;
    public static void main(String[] args) throws IOException {
        int servPort = 10000;
        // Create a server socket to accept client connection requests
        ServerSocket servSock = new ServerSocket(servPort);
        int recvMsgSize;   // Size of received message
        byte[] receiveBuf = new byte[BUFSIZE];
        while (true) {
            Socket clntSock = servSock.accept();
            SocketAddress clientAddress = clntSock.getRemoteSocketAddress();
            System.out.println("Handling client at " + clientAddress);
            InputStream in = clntSock.getInputStream();
            OutputStream out = clntSock.getOutputStream();
            // Receive until client closes connection, indicated by -1 return
            while ((recvMsgSize = in.read(receiveBuf)) != -1) {
                out.write(receiveBuf, 0, recvMsgSize);
            }
            clntSock.close();  // Close the socket.  We are done with this client!
        }
    }
}
```

Socket常用方法：

```java
void connect(); //打开一个TCP连接
InputStream getInputStream();
OutputStream getOutputStream();
void close(); // 关闭套接字及其输入输出流
void shutdownInput(); //关闭TCP流输入端，任何没有读取的数据都将被舍弃
void shutdownOutput();
// 获取 检测属性
InetAddress getInetAddress();
int getPort();
InetAddress getLocalAddress();
int getLocalPort();
SocketAddress getRemoteSocketAddress();
SocketAddress getLocalSocketAddress();
// ServerSocket 操作
void bind(int port); //为套接字关联一个本地端口, 只能关联一个本地端口
Socket accept(); // 为下一个传入的连接请求创建Socket实例，并返回
void close();
```

#### UDP套接字

UDP协议提供了一种不同于TCP协议的端到端服务。实际上UDP协议只实现两个功能：1）在IP协议的基础上添加了另一层地址（端口），2）对数据传输过程中可能产生的数据错误进行了检测，并抛弃已经损坏的数据。

**DatagramPacket**：Java程序员通过DatagramPacket 类和 DatagramSocket类来使用UDP套接字。客户端和服务器端都使用DatagramSockets来发送数据，使用DatagramPackets来接收数据。

**DatagramPacket**：UDP终端交换的是一种称为数据报文的自包含（self-contained）信息。这种信息在Java中表示为DatagramPacket类的实例，发送信息时，Java程序创建一个包含了待发送信息的DatagramPacket实例，并将其作为参数传递给DatagramSocket类的send()方法。接收信息时，Java程序首先创建一个DatagramPacket实例，该实例中预先分配了一些空间（一个字节数组byte[]），并将接收到的信息存放在该空间中。然后把该实例作为参数传递给DatagramSocket类的receive()方法。

客户端

```java
public class UDPEchoClientTimeout {
    private static final int TIMEOUT = 3000;
    private static final int MAXTRIES = 5;
    public static void main(String[] args) throws UnknownHostException, SocketException {
        String host = "localhost";
        int port = 10000;
        InetAddress serverAddress = InetAddress.getByName(host);
        byte[] bytesToSend  = "I'm UDP client".getBytes();
        DatagramSocket socket = new DatagramSocket();
        socket.setSoTimeout(TIMEOUT);
        DatagramPacket sendPacket = new DatagramPacket(bytesToSend, bytesToSend.length, serverAddress, port);
        DatagramPacket receivePacket = new DatagramPacket(new byte[bytesToSend.length], bytesToSend.length);
        int tries = 0;
        boolean received = false;
        while (!received && tries < MAXTRIES) {
            try {
                socket.send(sendPacket);
                socket.receive(receivePacket);
                if (!receivePacket.getAddress().equals(serverAddress)) {
                    throw new IOException("received packet from unknow source");
                }
                received = true;
            } catch (IOException e) {
                e.printStackTrace();
                tries++;
                System.out.println("Timed out, " + (MAXTRIES - tries) + " more tries");
            }
        }
        if (received) {
            System.out.println("received: " + new String(receivePacket.getData()));
        } else {
            System.out.println("No response after -- giving up");
        }
        socket.close();
    }
}
```

服务端：

```java
public class UDPEchoServer {
    private static final int ECHOMAX = 255;
    public static void main(String[] args) throws IOException {
        int port = 10000;
        DatagramSocket socket = new DatagramSocket(port);
        DatagramPacket packet = new DatagramPacket(new byte[ECHOMAX], ECHOMAX);
        while (true) {
            socket.receive(packet);
            System.out.println("Handling client at" + packet.getAddress().getHostAddress()
            + "on port " + packet.getPort());
            socket.send(packet);
            packet.setLength(ECHOMAX);
        }
    }
}
```

常用方法

```java
// DatagramPacket
InetAddress getAddress();
void setAddress();
int getPort();
void setPort();
void setSocketAddress();

int getLength();
void setLength();
int getOffset();
byte[] getData();
void setData(byte[] data);
// DatagramSocket
void send(DatagramPacket packet);
void receive(DatagramPacket packet);
int getSoTimeout();
void setSoTimeout(); // 设置receive方法调用的最长阻塞时间
```

#### 发送和接收数据

任何要交换信息的程序之间在信息的编码方式上必须达成共识（如将信息表示为位序列），以及哪个程序发送信息，什么时候和怎样接收信息都将影响程序的行为。程序间达成的这种包含了信息交换的形式和意义的共识称为协议，用来实现特定应用程序的协议叫做应用程序协议。

大部分的应用程序协议是根据由字段序列组成的离散信息定义的，其中每个字段中都包含了一段以位序列编码的特定的信息。应用程序协议中明确定义了信息的发送者应该怎样排列和解释这些位序列，同时还要定义接收者应该怎样解析，这样才使信息的接收者能够抽取出每个字段的意义。**TCP/IP协议的唯一约束是，信息必须在块（chunks）中发送和接收，而块的长度必须是8位的倍数**，因此，我们可以认为在TCP/IP协议中传输的信息是字节序列。鉴于此，我们可以进一步把传输的信息看作数字序列或数组，每个数字的取值范围是0到255。

**信息编码**

OutputStream、InputStream、DatagramPacket实例中所能处理的唯一数据类型是字节和字节数组。作为一种强类型语言，Java需要把其他数据类型（int，String等）显式转换成字节数组。

基本整型：1）定义要传输的每个整数的大小；2）字节的发送顺序；3）所传输的数值是有符号的还是无符号的；

字符串和文本：字符集编码达成共识；

组合输入输出流：Java中与流相关的类可以组合起来从而提供强大的功能。

```java
DataOutputStream out = new DataOutputStream( new BufferedOutputStream(socket.getOutputStream()));
```

**成帧和解析**

将数据转换成在线路上传输的格式只完成了一半工作，在接收端还必须将接收到的字节序列还原成原始信息。成帧（Framing）技术则解决了接收端如何定位消息的首尾位置的问题。无论信息是编码成了文本、多字节二进制数、或是两者的结合，应用程序协议必须指定消息的接收者如何确定何时消息已完整接收。

主要有两个技术使接收者能够准确地找到消息的结束位置：
(1)基于定界符（Delimiter-based）：消息的结束由一个唯一的标记（unique marker,）指出，即发送者在传输完数据后显式添加的一个特殊字节序列。这个特殊标记不能在传输的数据中出现。

(2)显式长度（Explicit length）：在变长字段或消息前附加一个固定大小的字段，用来指示该字段或消息中包含了多少字节。

**Java特定编码**

当使用套接字时，通常要么是需要同时创建通信信道两端的程序，要么实现一个给定的协议进行通信。如果知道通信双方都使用Java，并且你拥有对协议的完全控制权，那么就可以使用Java内置工具如Serializable接口或远程方法调用RMI。RMI能够调用不同Java虚拟机上的方法，并隐藏了所有繁琐的参数编码解码细节；序列化处理将Java对象转换成字节序列，你可以在不同虚拟机之间传递Java对象实例。

RMI的缺点：一个对象的序列化形式，在JVM之外的环境毫无意义；Serializable接口不能用于已经定义了不同传输格式的情况。所以，有些时候，实现自己的方法可能更简单、容易或更有效。

## 进阶



## 初识NIO



## 深入剖析





参考：

[Java TCP/IP Socket编程](https://book.douban.com/subject/3519369/)

[Java 网络编程](https://book.douban.com/subject/1438754/)

[UNIX网络编程 卷一：套接字编程](https://book.douban.com/subject/4859464/)

[Linux Socket编程（不限Linux）](https://www.cnblogs.com/skynet/archive/2010/12/12/1903949.html)