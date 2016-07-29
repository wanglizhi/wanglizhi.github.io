---
layout:     post
title:      "ThreadLocal那点事儿"
subtitle:   "ThreadLocal"
date:       2016-07-18 1:00:00
author:     "Wanglizhi"
header-img: "img/home-bg.jpg"
catalog:    true
tags:

    - Java
    - Java并发
---

## ThreadLocal详解

ThreadLocal，直译为“线程本地”或“本地线程”，如果你真的这么认为，那就错了！其实，它就是一个容器，用于存放线程的局部变量，我认为应该叫做 ThreadLocalVariable（线程局部变量）才对，真不理解为什么当初 Sun 公司的工程师这样命名。

早在 JDK 1.2 的时代，java.lang.ThreadLocal 就诞生了，它是为了解决多线程并发问题而设计的，只不过设计得有些难用，所以至今没有得到广泛使用。其实它还是挺有用的，不相信的话，我们一起来看看这个例子吧。

#### 不使用ThreadLocal的示例

一个序列号生成器的程序，可能同时会有多个线程并发访问它，要保证每个线程得到的序列号都是自增的，而不能相互干扰。

先定义一个接口：

```java
public interface Sequence {

    int getNumber();
}
```

每次调用 getNumber() 方法可获取一个序列号，下次再调用时，序列号会自增。

再做一个线程类：

```java
public class ClientThread extends Thread {

    private Sequence sequence;

    public ClientThread(Sequence sequence) {
        this.sequence = sequence;
    }

    @Override
    public void run() {
        for (int i = 0; i < 3; i++) {
            System.out.println(Thread.currentThread().getName() + " => " + sequence.getNumber());
        }
    }
}
```

在线程中连续输出三次线程名与其对应的序列号。我们先不用 ThreadLocal，来做一个实现类吧。

```java
public class SequenceA implements Sequence {

    private static int number = 0;

    public int getNumber() {
        number = number + 1;
        return number;
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceA();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

序列号初始值是0，在 main() 方法中模拟了三个线程，运行后结果如下：

Thread-0 => 1
Thread-0 => 2
Thread-0 => 3
Thread-2 => 4
Thread-2 => 5
Thread-2 => 6
Thread-1 => 7
Thread-1 => 8
Thread-1 => 9

由于线程启动顺序是随机的，所以并不是0、1、2这样的顺序，这个好理解。为什么当 Thread-0 输出了1、2、3之后，而 Thread-2 却输出了4、5、6呢？线程之间竟然共享了 static 变量！这就是所谓的“非线程安全”问题了。

#### 使用ThreadLocal保证线程安全

那么如何来保证“线程安全”呢？对应于这个案例，就是说不同的线程可拥有自己的 static 变量，如何实现呢？下面看看另外一个实现吧。

```java
public class SequenceB implements Sequence {

    private static ThreadLocal<Integer> numberContainer = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    public int getNumber() {
        numberContainer.set(numberContainer.get() + 1);
        return numberContainer.get();
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceB();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

通过 ThreadLocal 封装了一个 Integer 类型的 numberContainer 静态成员变量，并且初始值是0。再看 getNumber() 方法，首先从 numberContainer 中 get 出当前的值，加1，随后 set 到 numberContainer 中，最后将 numberContainer 中 get 出当前的值并返回。

运行结果如何呢？看看吧。

Thread-0 => 1
Thread-0 => 2
Thread-0 => 3
Thread-2 => 1
Thread-2 => 2
Thread-2 => 3
Thread-1 => 1
Thread-1 => 2
Thread-1 => 3

每个线程相互独立了，同样是 static 变量，对于不同的线程而言，它没有被共享，而是每个线程各一份，这样也就保证了线程安全。 也就是说，TheadLocal 为每一个线程提供了一个独立的副本！

搞清楚 ThreadLocal 的原理之后，有必要总结一下 ThreadLocal 的 API，其实很简单。

1. public void set(T value)：将值放入线程局部变量中
2. public T get()：从线程局部变量中获取值
3. public void remove()：从线程局部变量中移除值（有助于 JVM 垃圾回收）
4. protected T initialValue()：返回线程局部变量中的初始值（默认为 null） 

为什么 initialValue() 方法是 protected 的呢？就是为了提醒程序员们，这个方法是要你们来实现的，请给这个线程局部变量一个初始值吧。

#### 自己动手写ThreadLocal

了解了原理与这些 API，其实想想 ThreadLocal 里面不就是封装了一个 Map 吗？自己都可以写一个 ThreadLocal 了，尝试一下吧。

```java
public class MyThreadLocal<T> {

    private Map<Thread, T> container = Collections.synchronizedMap(new HashMap<Thread, T>());

    public void set(T value) {
        container.put(Thread.currentThread(), value);
    }

    public T get() {
        Thread thread = Thread.currentThread();
        T value = container.get(thread);
        if (value == null && !container.containsKey(thread)) {
            value = initialValue();
            container.put(thread, value);
        }
        return value;
    }

    public void remove() {
        container.remove(Thread.currentThread());
    }

    protected T initialValue() {
        return null;
    }
}
```

以上完全山寨了一个 ThreadLocal，其中中定义了一个同步 Map（为什么要这样？请读者自行思考），代码应该非常容易读懂。下面用这 MyThreadLocal 再来实现一把看看。

```java
public class SequenceC implements Sequence {

    private static MyThreadLocal<Integer> numberContainer = new MyThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    public int getNumber() {
        numberContainer.set(numberContainer.get() + 1);
        return numberContainer.get();
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceC();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

以上代码其实就是将 ThreadLocal 替换成了 MyThreadLocal，仅此而已，运行效果和之前的一样，也是正确的。ThreadLocal 具体有哪些使用案例呢？

我想首先要说的就是：通过 ThreadLocal 存放 JDBC Connection，以达到事务控制的能力。

## ThreadLocal应用

**注意：该示例仅用于说明 TheadLocal 的基本用法。在实际工作中，推荐使用连接池来管理数据库连接。示例中的代码仅作参考，使用前请酌情考虑。**

用户提出一个需求：当修改产品价格的时候，需要记录操作日志，什么时候做了什么事情。

```java
public class DBUtil {
    // 数据库配置
    private static final String driver = "com.mysql.jdbc.Driver";
    private static final String url = "jdbc:mysql://localhost:3306/demo";
    private static final String username = "root";
    private static final String password = "root";

    // 定义一个数据库连接
    private static Connection conn = null;

    // 获取连接
    public static Connection getConnection() {
        try {
            Class.forName(driver);
            conn = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return conn;
    }

    // 关闭连接
    public static void closeConnection() {
        try {
            if (conn != null) {
                conn.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

其实业务逻辑也不太复杂，于是我快速地完成了 ProductService 接口的实现类：

```java
public class ProductServiceImpl implements ProductService {

    private static final String UPDATE_PRODUCT_SQL = "update product set price = ? where id = ?";
    private static final String INSERT_LOG_SQL = "insert into log (created, description) values (?, ?)";

    public void updateProductPrice(long productId, int price) {
        try {
            // 获取连接
            Connection conn = DBUtil.getConnection();
            conn.setAutoCommit(false); // 关闭自动提交事务（开启事务）

            // 执行操作
            updateProduct(conn, UPDATE_PRODUCT_SQL, productId, price); // 更新产品
            insertLog(conn, INSERT_LOG_SQL, "Create product."); // 插入日志

            // 提交事务
            conn.commit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭连接
            DBUtil.closeConnection();
        }
    }

    private void updateProduct(Connection conn, String updateProductSQL, long productId, int productPrice) throws Exception {
        PreparedStatement pstmt = conn.prepareStatement(updateProductSQL);
        pstmt.setInt(1, productPrice);
        pstmt.setLong(2, productId);
        int rows = pstmt.executeUpdate();
        if (rows != 0) {
            System.out.println("Update product success!");
        }
    }

    private void insertLog(Connection conn, String insertLogSQL, String logDescription) throws Exception {
        PreparedStatement pstmt = conn.prepareStatement(insertLogSQL);
        pstmt.setString(1, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS").format(new Date()));
        pstmt.setString(2, logDescription);
        int rows = pstmt.executeUpdate();
        if (rows != 0) {
            System.out.println("Insert log success!");
        }
    }
}
```

竟然在多线程的环境下报错了，果然是数据库连接关闭了。怎么回事呢？既然是跟 Connection 有关系，那我就将主要精力放在检查 Connection 相关的代码上吧。是不是 Connection 不应该是 static 的呢？我当初设计成 static 的主要是为了让 DBUtil 的 static 方法访问起来更加方便，用 static 变量来存放 Connection 也提高了性能啊。

原来要使每个线程都拥有自己的连接，而不是共享同一个连接，否则线程1有可能会关闭线程2的连接，所以线程2就报错了。一定是这样！

```java
public class DBUtil {
    // 数据库配置
    private static final String driver = "com.mysql.jdbc.Driver";
    private static final String url = "jdbc:mysql://localhost:3306/demo";
    private static final String username = "root";
    private static final String password = "root";

    // 定义一个用于放置数据库连接的局部线程变量（使每个线程都拥有自己的连接）
    private static ThreadLocal<Connection> connContainer = new ThreadLocal<Connection>();

    // 获取连接
    public static Connection getConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn == null) {
                Class.forName(driver);
                conn = DriverManager.getConnection(url, username, password);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connContainer.set(conn);
        }
        return conn;
    }

    // 关闭连接
    public static void closeConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn != null) {
                conn.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connContainer.remove();
        }
    }
}
```

我把 Connection 放到了 ThreadLocal 中，这样每个线程之间就隔离了，不会相互干扰了。



参考：[ThreadLocal 那点事儿](http://my.oschina.net/huangyong/blog/159489)