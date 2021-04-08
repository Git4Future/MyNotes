# 每天一个知识点

## 1.RPC和HTTP请求区别？

**RPC:**

IPC (进程间通信) ，是在多任务操作系统或联网的计算机之间运行的程序和进程所用的通信技术。有两种类型的进程间通信（IPC）。
LPC(本地过程调用)，LPC用在多任务操作系统中，使得同时运行的任务能互相会话。这些任务共享内存空间使任务同步和互相发送信息。
RPC类似于LPC，只是在网上工作。

RPC，即 Remote Procedure Call（远程过程调用），是一个计算机**通信协议**。 其调用协议通常包含**传输协议**和**序列化协议**。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。说得通俗一点就是：A计算机提供一个服务，B计算机可以像调用本地服务那样调用A计算机的服务。

**RPC按传输协议:** 可以分为基于HTTP的、基于TCP等。如著名的 [gRPC](grpc / grpc.io) 使用的 http2 协议，也有如dubbo一类的自定义报文的tcp协议。

**RPC按报文协议:** 可以分为基于XML文本的、基于JSON文本的，二进制的。

**RPC按是否跨平台语言:**可以分为平台专用的，平台中立的。

**补充**: **WebService一般属于基于HTTP的、XML文本的、跨平台（平台中立）的RPC协议。功能完善、体系成熟、支持事务、支持安全机制，广泛应用在金融电信（中国电信一个省级分公司就有几千个WS），传统企业的业务系统，ESB/SOA体系等。缺点：过于复杂，性能不是最优的，互联网用的较少。**

**HTTP:**

Http(超文本传输协议)，是一种应用层协议。规定了网络传输的请求格式、响应格式、资源定位和操作的方式等。但是底层采用什么网络传输协议，并没有规定，不过现在都是采用TCP协议作为底层传输协议。

**HTTP和RPC的选用:**

速度来看，RPC要比http更快，虽然底层都是TCP，但是http协议的信息往往比较臃肿。所谓的效率优势是针对http1.1协议来讲的，http2.0协议已经优化编码效率问题，像grpc这种rpc库使用的就是http2.0协议。http容器的性能测试单位通常是kqps，自定义tpc协议则通常是10kqps到100kqps。

难度来看，RPC实现较为复杂，http相对比较简单。RPC需要满足像调用本地服务一样调用远程服务，也就是对调用过程在API层面进行封装。Http协议没有这样的要求，因此请求、响应等细节需要我们自己去实现。

- 优点：RPC方式更加透明，对用户更方便。Http方式更灵活，没有规定API和语言，跨语言、跨平台
- 缺点：RPC方式需要在API层面进行封装，限制了开发的语言环境。调用端和服务端需要使用相同的技术，要么都hessian，要么都dubbo。

**总结:成熟的rpc库相对http容器，更多的是封装了“服务发现”，"负载均衡"，“熔断降级”一类面向服务的高级特性。可以这么理解，rpc框架是面向服务的更高级的封装。如果把一个http servlet容器上封装一层服务发现和函数代理调用，那它就已经可以做一个rpc框架了。**



## 2.cookie、session和token的区别?

**关于session:**
用户登录网页时会给服务器发送请求，带着用户信息，此时服务器会进行判断并返回session给用户

1.服务器获取用户信息。
2.判断验证用户。
3.判断成功，将用户信息写入redis并得到sessionid。
4.将session写入cookie返回给前端发送给用户。
5.用户拿着session去请求服务器，服务器解析session后判断用户信息并成功访问。

**session的缺点：**
1.因为用户信息是保存在**服务端**的**内存中的**，随着用户量增大，所需要的内存资源也随之增大，不可避免的造成内存溢出。
2.因为session是基于cookie来进行用户识别的，所以如果cookie会被黑客截获进行CSRF攻击，这样一来就容易被跨站攻击，用户信息被泄露，造成经济损失,虽然   可以关闭浏览器每次访问都携带cookie和session信息的功能，但是有些网站是基于session登录，这样一来就无法使用该网页。
3.随着用户量信息量增多，不可避免进行扩容，建立服务器集群。而随之而来问题就是当用户登录服务器1的时候进行下一个操作可能会被分配到服务器2，此时服务器2没有保存用户的登录信息，所以需要进行再次登录。此时造成了逻辑缺失。

**关于token**
token机制与session机制的差距不大，最主要的是服务器处理的第三步：

session： 判断成功，将用户信息写入redis并得到sessionid。
token：判断成功，将用户信息进行加密生成一个加密字符串保存在token变量中。
此时服务器返回给用户的就不是session值而是一个token值,前端接受加密字符串通过js代码保存在storage中，用户使用该token值访问网页，网页使用get方法通过js代码获取storage中的token值，并且进行解密，当解密成功获取用户信息。

token值就像一串随机字符串，就算被黑客截获也无法使用token来跨域攻击网站致使用户信息泄露，因为浏览器的同源策略致使无法使用不同浏览器登录用户；其次token值不是存在cookie中，而是随着js代码保存在storage中。



## 3.== 和equals的区别?

1. **==，比较的是值是否相等。**

如果作用于基本数据类型的变量，则直接比较其存储的 值是否相等，如果作用于引用类型的变量，则比较的是所指向的对象的地址是否相等。

```
其实==比较的不管是基本数据类型，还是引用数据类型的变量，比较的都是值，只是引用类型变量存的值是对象的地址
```

2. **对于equals方法，比较的是是否是同一个对象。**

首先，equals()方法不能作用于基本数据类型的变量，

另外，equals()方法存在于Object类中，而Object类是所有类的直接或间接父类，所以说所有类中的equals()方法都继承自Object类，在没有重写equals()方法的类中，调用equals()方法其实和使用==的效果一样，也是比较的是引用类型的变量所指向的对象的地址，不过，Java提供的类中，有些类都重写了equals()方法，重写后的equals()方法一般都是比较两个对象的值，比如String类。

Object类equals()方法源码：

```
public boolean equals(Object obj) {
     return (this == obj);
}
```

- 情况 1：类没有覆盖 `equals()`方法。则通过`equals()`比较该类的两个对象时，等价于通过“==”比较这两个对象。使用的默认是 `Object`类`equals()`方法。
- 情况 2：类覆盖了 `equals()`方法。一般，我们都覆盖 `equals()`方法来两个对象的内容相等；若它们的内容相等，则返回 true(即，认为这两个对象相等)。

```java
public class test1 {
    public static void main(String[] args) {
        String a = new String("ab"); // a 为一个引用
        String b = new String("ab"); // b为另一个引用,对象的内容一样
        String aa = "ab"; // 放在常量池中
        String bb = "ab"; // 从常量池中查找
        if (aa == bb) // true
            System.out.println("aa==bb");
        if (a == b) // false，非同一对象
            System.out.println("a==b");
        if (a.equals(b)) // true
            System.out.println("aEQb");
        if (42 == 42.0) { // true
            System.out.println("true");
        }
    }
}
```

**说明：**

- `String` 中的 `equals` 方法是被重写过的，因为 `Object` 的 `equals` 方法是比较的对象的内存地址，而 `String` 的 `equals` 方法比较的是对象的值。
- 当创建 `String` 类型的对象时，虚拟机会在常量池中查找有没有已经存在的值和要创建的值相同的对象，如果有就把它赋给当前引用。如果没有就在常量池中重新创建一个 `String` 对象。

`String`类`equals()`方法：

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```