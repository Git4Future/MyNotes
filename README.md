# 常用框架

## Spring相关问题

### 1.容器系列的设计与实现：BeanFactory和ApplicationContext

在Spring IoC容器的设计中，我们可以看到两个主要的容器系列，一个是实现BeanFactory接口的简单容器系列，这系列容器只实现了容器的最基本功能；另一个是ApplicationContext应用上下文，它作为容器的高级形态而存在。应用上下文在简单容器的基础上，增加了许多面向框架的特性，同时对应用环境作了许多适配。在这些Spring提供的基本IoC容器的接口定义和实现的基础上，Spring通过定义BeanDefinition来管理基于Spring的应用中的各种对象以及它们之间的相互依赖关系。BeanDefinition抽象了我们对Bean的定义，是让容器起作用的主要数据类型。

![1568098087203](/home/anyuan/.config/Typora/typora-user-images/1568098087203.png)

从接口BeanFactory到HierarchicalBeanFactory，再到ConfigurableBeanFactory，是一条主要的BeanFactory设计路径。在这条接口设计路径中，BeanFactory接口定义了基本的IoC容器的规范。在这个接口定义中，包括了getBean()这样的IoC容器的基本方法（通过这个方法可以从容器中取得Bean）。而HierarchicalBeanFactory接口在继承了BeanFactory的基本接口之后，增加了getParentBeanFactory()的接口功能，使BeanFactory具备了双亲IoC容器的管理功能。在接下来的ConfigurableBeanFactory接口中，主要定义了一些对BeanFactory的配置功能，比如通过setParentBeanFactory()设置双亲IoC容器，通过addBeanPostProcessor()配置Bean后置处理器，等等。通过这些接口设计的叠加，定义了BeanFactory就是简单IoC容器的基本功能。

第二条接口设计主线是，以ApplicationContext应用上下文接口为核心的接口设计，这里涉及的主要接口设计有，从BeanFactory到ListableBeanFactory，再到ApplicationContext，再到我们常用的WebApplicationContext或者ConfigurableApplicationContext接口。我们常用的应用上下文基本上都是ConfigurableApplicationContext或者WebApplicationContext的实现。在这个接口体系中，ListableBeanFactory和HierarchicalBeanFactory两个接口，连接BeanFactory接口定义和ApplicationConext应用上下文的接口定义。在ListableBeanFactory接口中，细化了许多BeanFactory的接口功能，比如定义了getBeanDefinitionNames()接口方法；对于HierarchicalBeanFactory接口，我们在前文中已经提到过；对于ApplicationContext接口，它通过继承MessageSource、ResourceLoader、ApplicationEventPublisher接口，在BeanFactory简单IoC容器的基础上添加了许多对高级容器的特性的支持。



### 2.IOC容器的初始化

简单来说，IoC容器的初始化是由前面介绍的refresh()方法来启动的，这个方法标志着IoC容器的正式启动。具体来说，这个启动包括BeanDefinition的Resouce定位、载入和注册三个基本过程。
***第一个过程是Resource定位过程。这个Resource定位指的是BeanDefinition的资源定位，它由ResourceLoader通过统一的Resource接口来完成，这个Resource对各种形式的BeanDefinition的使用都提供了统一接口。***对于这些BeanDefinition的存在形式，相信大家都不会感到陌生。比如，在文件系统中的Bean定义信息可以使用FileSystemResource来进行抽象；在类路径中的Bean定义信息可以使用前面提到的ClassPathResource来使用.
***第二个过程是BeanDefinition的载入。这个载入过程是把用户定义好的Bean表示成IoC容器内部的数据结构，而这个容器内部的数据结构就是BeanDefinition***。具体来说，这个BeanDefinition实际上就是POJO对象在IoC容器中的抽象，通过这个BeanDefinition定义的数据结构，使IoC容器能够方便地对POJO对象也就是Bean进行管理。
***第三个过程是向IoC容器注册这些BeanDefinition的过程。这个过程是通过调用BeanDefinitionRegistry接口的实现来完成的***。这个注册过程把载入过程中解析得到的BeanDefinition向IoC容器进行注册。通过分析，我们可以看到，在IoC容器内部将BeanDefinition注入到一个HashMap中去，IoC容器就是通过这个HashMap来持有这些BeanDefinition数据的。
值得注意的是，这里谈的是IoC容器初始化过程，在这个过程中，一般不包含Bean依赖注入的实现。在Spring IoC的设计中，Bean定义的载入和依赖注入是两个独立的过程。依赖注入一般发生在应用第一次通过getBean向容器索取Bean的时候。但有一个例外值得注意，在使用IoC容器时有一个预实例化的配置，通过这个预实例化的配置（具体来说，可以通过为Bean定义信息中的lazyinit属性），用户可以对容器初始化过程作一个微小的控制，从而改变这个被设置了lazyinit属性的Bean的依赖注入过程。举例来说，如果我们对某个Bean设置了lazyinit属性，那么这个Bean的依赖注入在IoC容器初始化时就预先完成了，而不需要等到整个初始化完成以后，第一次使用getBean时才会触发。

#### **BeanDefinition的Resource定位**

以编程的方式使用DefaultListableBeanFactory时，首先定义一个Resource来定位容器使用的BeanDefinition。这时使用的是ClassPathResource，这意味着Spring会在类路径中去寻找以文件形式存在的BeanDefinition信息。
ClassPathResource res = new ClassPathResource("beans.xml");
这里定义的Resource并不能由DefaultListableBeanFactory直接使用，Spring通过BeanDefinitionReader来对这些信息进行处理。在这里，我们也可以看到使用Application-Context相对于直接使用DefaultListableBeanFactory的好处。因为在ApplicationContext中，Spring已经为我们提供了一系列加载不同Resource的读取器的实现，而DefaultListableBeanFactory只是一个纯粹的IoC容器，需要为它配置特定的读取器才能完成这些功能。当然，有利就有弊，使用DefaultListableBeanFactory这种更底层的容器，能提高定制IoC容器的灵活性。
回到我们经常使用的ApplicationContext上来，例如FileSystemXmlApplicationContext、ClassPathXmlApplicationContext以及XmlWebApplicationContext等。简单地从这些类的名字上分析，可以清楚地看到它们可以提供哪些不同的Resource读入功能，比如FileSystemXmlApplicationContext可以从文件系统载入Resource，ClassPathXmlApplication-Context可以从Class Path载入Resource，XmlWebApplicationContext可以在Web容器中载入Resource，等等。

下面以FileSystemXmlApplicationContext为例，通过分析这个ApplicationContext的实现来看看它是怎样完成这个Resource定位过程的。作为辅助，我们可以在图2-5中看到相应的ApplicationContext继承体系。

![1568098786130](/home/anyuan/.config/Typora/typora-user-images/1568098786130.png)


从图2-6中可以看到，这个FileSystemXmlApplicationContext已经通过继承Abstract-ApplicationContext具备了ResourceLoader读入以Resource定义的BeanDefinition的能力，因为AbstractApplicationContext的基类是DefaultResourceLoader。

FileSystemApplicationContext是一个支持XML定义BeanDefinition的ApplicationContext，并且可以指定以文件形式的BeanDefinition的读入，这些文件可以使用文件路径和URL定义来表示。在测试环境和独立应用环境中，这个ApplicationContext是非常有用的。
根据图2-7的调用关系分析，我们可以清楚地看到整个BeanDefinition资源定位的过程。这个对BeanDefinition资源定位的过程，最初是由refresh来触发的，这个refresh的调用是在FileSystemXmlBeanFactory的构造函数中启动的，大致的调用过程如图2-8所示。
前面我们看到的getResourceByPath会被子类FileSystemXmlApplicationContext实现，这个方法返回的是一个 FileSystemResource对象，通过这个对象，Spring可以进行相关的I/O操作，完成BeanDefinition的定位。分析到这里已经一目了然，它实现的就是对path进行解析，然后生成一个FileSystemResource对象并返回，

如果是其他的ApplicationContext，那么会对应生成其他种类的Resource，比如ClassPathResource、ServletContextResource等。通过对前面的实现原理的分析，我们以FileSystemXmlApplicationContext的实现原理为例子，了解了Resource定位问题的解决方案，即以FileSystem方式存在的Resource的定位实现。在BeanDefinition定位完成的基础上，就可以通过返回的Resource对象来进行BeanDefinition的载入了。在定位过程完成以后，为BeanDefinition的载入创造了I/O操作的条件，但是具体的数据还没有开始读入。

#### **BeanDefinition的载入和解析**

通过以上对实现原理的分析，我们可以看到，在初始化FileSystmXmlApplicationContext的过程中是通过调用IoC容器的refresh来启动整个BeanDefinition的载入过程的，这个初始化是通过定义的XmlBeanDefinitionReader来完成的。同时，我们也知道实际使用的IoC容器是DefultListableBeanFactory，具体的Resource载入在XmlBeanDefinitionReader读入BeanDefinition时实现。因为Spring可以对应不同形式的BeanDefinition。由于这里使用的是XML方式的定义，所以需要使用XmlBeanDefinitionReader。如果使用了其他的BeanDefinition方式，就需要使用其他种类的BeanDefinitionReader来完成数据的载入工作。在XmlBeanDefinitionReader的实现中可以看到，是在reader.loadBeanDefinitions中开始进行BeanDefinition的载入的，
BeanDefinition的载入分成两部分，首先通过调用XML的解析器得到document对象，但这些document对象并没有按照Spring的Bean规则进行解析。在完成通用的XML解析以后，才是按照Spring的Bean规则进行解析的地方，这个按照Spring的Bean规则进行解析的过程是在 documentReader中实现的。这里使用的documentReader是默认设置好的DefaultBean-DefinitionDocumentReader。这个DefaultBeanDefinitionDocumentReader的创建是在后面的方法中完成的，然后再完成BeanDefinition的处理，处理的结果由BeanDefinitionHolder对象来持有。这个BeanDefinitionHolder除了持有BeanDefinition对象外，还持有其他与BeanDefinition的使用相关的信息，比如Bean的名字、别名集合等。这个BeanDefinition-Holder的生成是通过对Document文档树的内容进行解析来完成的，可以看到这个解析过程是由BeanDefinition-ParserDelegate来实现（具体在processBeanDefinition方法中实现）的。

#### **BeanDefinition在IoC容器中的注册**

前面已经分析过BeanDefinition在IoC容器中载入和解析的过程。在这些动作完成以后，用户定义BeanDefinition信息已经在IoC容器内建立起了自己的数据结构以及相应的数据表示，但此时这些数据还不能供IoC容器直接使用，需要在IoC容器中对这些BeanDefinition数据进行注册。这个注册为IoC容器提供了更友好的使用方式，在DefaultListableBeanFactory中，是通过一个HashMap来持有载入的BeanDefinition的，这个HashMap的定义在DefaultListableBeanFactory中可以看到，如下所示。

`/** Map of bean definition objects, keyed by bean name */
private final Map<String, BeanDefinition> beanDefinitionMap = new
ConcurrentHashMap<String, BeanDefinition>();`

### 3.Spring中的bean是线程安全的吗？

Spring容器中的Bean是否线程安全，容器本身并没有提供Bean的线程安全策略，因此可以说Spring容器中的Bean本身不具备线程安全的特性，但是具体还是要结合具体scope的Bean去研究。

Spring 的 bean 作用域（scope）类型
1、singleton:单例，默认作用域。
2、prototype:原型，每次创建一个新对象。
3、request:请求，每次Http请求创建一个新对象，适用于WebApplicationContext环境下。
4、session:会话，同一个会话共享一个实例，不同会话使用不用的实例。
5、global-session:全局会话，所有会话共享一个实例。
线程安全这个问题，要从单例与原型Bean分别进行说明。
原型Bean：
对于原型Bean,每次创建一个新对象，也就是线程之间并不存在Bean共享，自然是不会有线程安全的问题。

单例Bean：
对于单例Bean,所有线程都共享一个单例实例Bean,因此是存在资源的竞争。

如果单例Bean,是一个无状态Bean，也就是线程中的操作不会对Bean的成员执行查询以外的操作，那么这个单例Bean是线程安全的。比如Spring mvc 的 Controller、Service、Dao等，这些Bean大多是无状态的，只关注于方法本身。对于有状态的bean，Spring官方提供的bean，一般提供了通过ThreadLocal去解决线程安全的方法，比如RequestContextHolder、TransactionSynchronizationManager、LocaleContextHolder等。

使用ThreadLocal的好处
使得多线程场景下，多个线程对这个单例Bean的成员变量并不存在资源的竞争，因为ThreadLocal为每个线程保存线程私有的数据。这是一种以空间换时间的方式。当然也可以通过加锁的方法来解决线程安全，这种以时间换空间的场景在高并发场景下显然是不实际的。

### 4.@RestController vs @Controller?

**@controller返回一个页面**

单独使用 `@Controller` 不加 `@ResponseBody`的话一般使用在要返回一个视图的情况，这种情况属于比较传统的Spring MVC 的应用，对应于前后端不分离的情况。

**@RestController 返回JSON 或 XML 形式数据**

但`@RestController`只返回对象，对象数据直接以 JSON 或 XML 形式写入 HTTP 响应(Response)中，这种情况属于 RESTful Web服务，这也是目前日常开发所接触的最常用的情况（前后端分离）。

**@Controller +@ResponseBody 返回JSON 或 XML 形式数据**

如果你需要在Spring4之前开发 RESTful Web服务的话，你需要使用@Controller 并结合@ResponseBody注解，也就是说@Controller +@ResponseBody= @RestController（Spring 4 之后新加的注解）。

@ResponseBody 注解的作用是将 Controller 的方法返回的对象通过适当的转换器转换为指定的格式之后，写入到HTTP 响应(Response)对象的 body 中，通常用来返回 JSON 或者 XML 数据，返回 JSON 数据的情况比较多

### 5.Spring AOP 和 AspectJ AOP 有什么区别？

AOP(Aspect OrientedProgramming, 面向切面/方面编程) 旨在从业务逻辑中分离出来横切逻辑【eg:性能监控、日志记录、权限控制等】，提高模块化，即通过AOP解决代码耦合问题，让职责更加单一。

#### **运用技术：**

  SpringAOP使用了两种代理机制，一种是基于JDK的动态代理，另一种是基于CGLib的动态代理，之所以需要两种代理机制，很大程度上是因为JDK本身只提供基于接口的代理，不支持类的代理。

#### **切面植入的方法：**

​     **1、编译期织入**

​     **2、类装载期织入**

​     **3、动态代理织入---->**在运行期为目标类添加增强生成子类的方式，Spring AOP采用动态代理织入切面

#### **流行的框架：**

​    AOP现有两个主要的流行框架，即Spring AOP和Spring+AspectJ。

![img](https://img-blog.csdn.net/20160113113029764?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### **Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。** Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)。

Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。AspectJ 相比于 Spring AOP 功能更加强大，但是 Spring AOP 相对来说更简单，

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择 AspectJ ，它比Spring AOP 快很多。





# Java

## 并发

### 一、并发面试题总结

#### 1. 什么是线程和进程?

**进程是程序的一次执行过程**，是一个动态概念，是程序在执行过程中分配和管理资源的基本单位，每一个进程都有一个自己的地址空间，至少有 5 种基本状态，它们是：初始态，执行态，等待状态，就绪状态，终止状态。

**线程是CPU调度和分派的基本单位**，它可与同属一个进程的其他的线程**共享**进程所拥有的全部资源。

线程与进程相似，但线程是一个比进程更小的执行单位。一个进程在其执行的过程中可以产生多个线程。与进程不同的是同类的多个线程共享进程的**堆**和**方法区**资源，但每个线程有自己的**程序计数器**、**虚拟机栈**和**本地方法栈**，所以系统在产生一个线程，或是在各个线程之间作切换工作时，负担要比进程小得多，也正因为如此，线程也被称为轻量级进程。

Java 程序天生就是多线程程序，我们可以通过 JMX 来看一下一个普通的 Java 程序有哪些线程，代码如下。

```
public class MultiThread {
	public static void main(String[] args) {
		// 获取 Java 线程管理 MXBean
	ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
		// 不需要获取同步的 monitor 和 synchronizer 信息，仅获取线程和线程堆栈信息
		ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
		// 遍历线程信息，仅打印线程 ID 和线程名称信息
		for (ThreadInfo threadInfo : threadInfos) {
			System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.getThreadName());
		}
	}
}
```

上述程序输出如下（输出内容可能不同，不用太纠结下面每个线程的作用，只用知道 main 线程执行 main 方法即可）：

```Java
[5] Attach Listener //添加事件
[4] Signal Dispatcher // 分发处理给 JVM 信号的线程
[3] Finalizer //调用对象 finalize 方法的线程
[2] Reference Handler //清除 reference 线程
[1] main //main 线程,程序入口
```

从上面的输出内容可以看出：**一个 Java 程序的运行是 main 线程和多个其他线程同时运行**。

#### 2. 程序计数器为什么是私有的?

程序计数器主要有下面两个作用：

1. 字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
2. 在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。

需要注意的是，如果执行的是 native 方法，那么程序计数器记录的是 undefined 地址，只有执行的是 Java 代码时程序计数器记录的才是下一条指令的地址。

所以，程序计数器私有主要是为了**线程切换后能恢复到正确的执行位置**。

#### 3. 虚拟机栈和本地方法栈为什么是私有的?

- **虚拟机栈：** 每个 Java 方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。
- **本地方法栈：** 和虚拟机栈所发挥的作用非常相似，区别是： **虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

所以，为了**保证线程中的局部变量不被别的线程访问到**，虚拟机栈和本地方法栈是线程私有的。

#### 4. 为什么要使用多线程呢?

- **从计算机底层来说：** 线程可以比作是轻量级的进程，是程序执行的最小单位,线程间的切换和调度的成本远远小于进程。另外，多核 CPU 时代意味着多个线程可以同时运行，这减少了线程上下文切换的开销。
- **从当代互联网发展趋势来说：** 现在的系统动不动就要求百万级甚至千万级的并发量，而多线程并发编程正是开发高并发系统的基础，利用好多线程机制可以大大提高系统整体的并发能力以及性能
- **单核时代：** 在单核时代多线程主要是为了提高 CPU 和 IO 设备的综合利用率。举个例子：当只有一个线程的时候会导致 CPU 计算时，IO 设备空闲；进行 IO 操作时，CPU 空闲。我们可以简单地说这两者的利用率目前都是 50%左右。但是当有两个线程的时候就不一样了，当一个线程执行 CPU 计算时，另外一个线程可以进行 IO 操作，这样两个的利用率就可以在理想情况下达到 100%了。
- **多核时代:** 多核时代多线程主要是为了提高 CPU 利用率。举个例子：假如我们要计算一个复杂的任务，我们只用一个线程的话，CPU 只会一个 CPU 核心被利用到，而创建多个线程就可以让多个 CPU 核心被利用到，这样就提高了 CPU 的利用率。 

#### 5. 使用多线程可能带来什么问题?

1、线程过多：如果系统上的线程数量远远超过核心的数量，那么就会导致频繁的上下文切换，进而降低性能，如缓存污染。通常支持超线程的多核处理器能够使用的线程数最多是物理核心数的2倍，再增加就有可能降低程序的性能

2、数据竞争：当多个线程读写同一共享数据时，便会产生竞争，需要同步，同步通常会导致线程之间的相互等待，潜在的降低了性能；另一方面，如果不使用同步程序可能无法并行。

3、死锁：线程发生死锁时，处理器都在操作（一直询问需要的资源是否可用），但是线程都在相互等待其他线程释放资源，处于僵持状态。

4、饿死：当一个或多个线程永远没有机会调度到处理器上执行，而陷入永远的等待的状态。

5、伪共享：当多个线程读写的数据映射到同一条缓存线上时，如果一个线程更改了数据，那么其他线程对该数据的缓存就要被失效，，如果频繁地更改数据，硬件就需要不停的更新缓存线，这使性能从独享缓存的水平降低到共享缓存或内存的水平。

#### 6. 说说线程的生命周期和状态?


Java 线程在运行的生命周期中的指定时刻只可能处于下面 6 种不同状态的其中一个状态（图源《Java 并发编程艺术》4.1.4 节）。

[![Java 线程的状态 ](https://camo.githubusercontent.com/bd21f0c6bf04fe410fa5397897cc47b9278ae5cb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31392d312d32392f4a6176612545372542412542462545372541382538422545372539412538342545372538412542362545362538302538312e706e67)](https://camo.githubusercontent.com/bd21f0c6bf04fe410fa5397897cc47b9278ae5cb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31392d312d32392f4a6176612545372542412542462545372541382538422545372539412538342545372538412542362545362538302538312e706e67)

线程在生命周期中并不是固定处于某一个状态而是随着代码的执行在不同状态之间切换。Java 线程状态变迁如下图所示（图源《Java 并发编程艺术》4.1.4 节）：

[![Java 线程状态变迁 ](https://camo.githubusercontent.com/e518e038e37c2d27abb394b00b438d347466c90c/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31392d312d32392f4a6176612b2545372542412542462545372541382538422545372538412542362545362538302538312545352538462539382545382542462538312e706e67)](https://camo.githubusercontent.com/e518e038e37c2d27abb394b00b438d347466c90c/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31392d312d32392f4a6176612b2545372542412542462545372541382538422545372538412542362545362538302538312545352538462539382545382542462538312e706e67)

由上图可以看出：线程创建之后它将处于 **NEW（新建）** 状态，调用 `start()` 方法后开始运行，线程这时候处于 **READY（可运行）** 状态。可运行状态的线程获得了 CPU 时间片（timeslice）后就处于 **RUNNING（运行）** 状态。

> 操作系统隐藏 Java 虚拟机（JVM）中的 RUNNABLE 和 RUNNING 状态，它只能看到 RUNNABLE 状态（图源：[HowToDoInJava](https://howtodoinjava.com/)：[Java Thread Life Cycle and Thread States](https://howtodoinjava.com/java/multi-threading/java-thread-life-cycle-and-thread-states/)），所以 Java 系统一般将这两个状态统称为 **RUNNABLE（运行中）** 状态 。

[![RUNNABLE-VS-RUNNING](https://camo.githubusercontent.com/916fefa029894b21921d3085f513b9a7f08ebad2/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d332f52554e4e41424c452d56532d52554e4e494e472e706e67)](https://camo.githubusercontent.com/916fefa029894b21921d3085f513b9a7f08ebad2/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d332f52554e4e41424c452d56532d52554e4e494e472e706e67)

当线程执行 `wait()`方法之后，线程进入 **WAITING（等待）** 状态。进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而 **TIME_WAITING(超时等待)** 状态相当于在等待状态的基础上增加了超时限制，比如通过 `sleep（long millis）`方法或 `wait（long millis）`方法可以将 Java 线程置于 TIMED WAITING 状态。当超时时间到达后 Java 线程将会返回到 RUNNABLE 状态。当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到 **BLOCKED（阻塞）** 状态。线程在执行 Runnable 的`run()`方法之后将会进入到 **TERMINATED（终止）** 状态。

#### 7. 什么是上下文切换?

多线程编程中一般线程的个数都大于 CPU 核心的个数，而一个 CPU 核心在任意时刻只能被一个线程使用，为了让这些线程都能得到有效执行，CPU 采取的策略是为每个线程分配时间片并轮转的形式。当一个线程的时间片用完的时候就会重新处于就绪状态让给其他线程使用，这个过程就属于一次上下文切换。

概括来说就是：当前任务在执行完 CPU 时间片切换到另一个任务之前会先保存自己的状态，以便下次再切换回这个任务时，可以再加载这个任务的状态。**任务从保存到再加载的过程就是一次上下文切换**。

上下文切换通常是计算密集型的。也就是说，它需要相当可观的处理器时间，在每秒几十上百次的切换中，每次切换都需要纳秒量级的时间。所以，上下文切换对系统来说意味着消耗大量的 CPU 时间，事实上，可能是操作系统中时间消耗最大的操作。

Linux 相比与其他操作系统（包括其他类 Unix 系统）有很多的优点，其中有一项就是，其上下文切换和模式切换的时间消耗非常少。

#### 8. 什么是线程死锁?如何避免死锁?

多个线程同时被阻塞，它们中的一个或者全部都在等待某个资源被释放。由于线程被无限期地阻塞，因此程序不可能正常终止。

如下图所示，线程 A 持有资源 2，线程 B 持有资源 1，他们同时都想申请对方的资源，所以这两个线程就会互相等待而进入死锁状态。



[![线程死锁示意图 ](https://camo.githubusercontent.com/3903a4dc24008be52f72bad23498808b5a743c35/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d342f323031392d34254536254144254242254539253934253831312e706e67)](https://camo.githubusercontent.com/3903a4dc24008be52f72bad23498808b5a743c35/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d342f323031392d34254536254144254242254539253934253831312e706e67)

学过操作系统的朋友都知道产生死锁必须具备以下四个条件：

1. 互斥条件：该资源任意一个时刻只由一个线程占用。
2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
3. 不剥夺条件:线程已获得的资源在末使用完之前不能被其他线程强行剥夺，只有自己使用完毕后才释放资源。
4. 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

##### 如何避免线程死锁?

我们只要破坏产生死锁的四个条件中的其中一个就可以了。

**破坏互斥条件**

这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。

**破坏请求与保持条件**

一次性申请所有的资源。

**破坏不剥夺条件**

占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

**破坏循环等待条件**

靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。

#### 9. 说说 sleep() 方法和 wait() 方法区别和共同点?

- 两者最主要的区别在于：**sleep 方法没有释放锁，而 wait 方法释放了锁** 。
- 两者都可以暂停线程的执行。
- Wait 通常被用于线程间交互/通信，sleep 通常被用于暂停执行。
- wait() 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 notify() 或者 notifyAll() 方法。sleep() 方法执行完成后，线程会自动苏醒。或者可以使用wait(long timeout)超时后线程会自动苏醒。

#### 10. 为什么我们调用 start() 方法时会执行 run() 方法，为什么我们不能直接调用 run() 方法？

new 一个 Thread，线程进入了新建状态;调用 start() 方法，会启动一个线程并使线程进入了就绪状态，当分配到时间片后就可以开始运行了。 start() 会执行线程的相应准备工作，然后自动执行 run() 方法的内容，这是真正的多线程工作。 而直接执行 run() 方法，会把 run 方法当成一个 main 线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。

**总结： 调用 start 方法方可启动线程并使线程进入就绪状态，而 run 方法只是 thread 的一个普通方法调用，还是在主线程里执行。**

#### 11. 说一说自己对于 synchronized 关键字的了解

synchronized关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。

另外，在 Java 早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的 Mutex Lock(互斥锁) 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的 synchronized 效率低的原因。Java 6 之后 Java 官方对从 JVM 层面对synchronized 较大优化，所以现在的 synchronized 锁效率也优化得很不错了。JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

#### 12.synchronized关键字最主要的三种使用方式

- **修饰实例方法:** 作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁

- **修饰静态方法:** :也就是给当前类加锁，会作用于类的所有对象实例，因为静态成员不属于任何一个实例对象，是类成员（ static 表明这是该类的一个静态资源，不管new了多少个对象，只有一份）。所以如果一个线程A调用一个实例对象的非静态 synchronized 方法，而线程B需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，**因为访问静态 synchronized 方法占用的锁是当前类的锁，而访问非静态 synchronized 方法占用的锁是当前实例对象锁**。

- **修饰代码块:** 指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

  **总结：** synchronized 关键字加到 static 静态方法和 synchronized(class)代码块上都是是给 Class 类上锁。synchronized 关键字加到静态方法上是给对象实例上锁。尽量不要使用 synchronized(String a) 因为JVM中，字符串常量池具有缓存功能！

  **双重校验锁实现对象单例（线程安全）**

  ```Java
  public class Singleton {
  
      private volatile static Singleton uniqueInstance;
  
      private Singleton() {
      }
  
      public static Singleton getUniqueInstance() {
         //先判断对象是否已经实例过，没有实例化过才进入加锁代码
          if (uniqueInstance == null) {
              //类对象加锁
              synchronized (Singleton.class) {
                  if (uniqueInstance == null) {
                      uniqueInstance = new Singleton();
                  }
              }
          }
          return uniqueInstance;
      }
  }
  ```

  另外，需要注意 uniqueInstance 采用 volatile 关键字修饰也是很有必要。

  uniqueInstance 采用 volatile 关键字修饰也是很有必要的， uniqueInstance = new Singleton(); 这段代码其实是分为三步执行：

  1. 为 uniqueInstance 分配内存空间
  2. 初始化 uniqueInstance
  3. 将 uniqueInstance 指向分配的内存地址

  但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。指令重排在单线程环境下不会出先问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程 T1 执行了 1 和 3，此时 T2 调用 getUniqueInstance() 后发现 uniqueInstance 不为空，因此返回 uniqueInstance，但此时 uniqueInstance 还未被初始化。

  使用 volatile 可以禁止 JVM 的指令重排，保证在多线程环境下也能正常运行。

#### 13. synchronized 关键字的底层原理

​     **synchronized 关键字底层原理属于 JVM 层面。**

**① synchronized 同步语句块的情况**

**synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。** 当执行 monitorenter 指令时，线程试图获取锁也就是获取 monitor(monitor对象存在于每个Java对象的对象头中，synchronized 锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因) 的持有权。当计数器为0则可以成功获取，获取后将锁计数器设为1也就是加1。相应的在执行 monitorexit 指令后，将锁计数器设为0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

**② synchronized 修饰方法的的情况**

synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

#### 14.说说 JDK1.6 之后的synchronized 关键字底层做了哪些优化

JDK1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

锁主要存在四中状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

**①偏向锁**

**引入偏向锁的目的和引入轻量级锁的目的很像，他们都是为了没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。但是不同是：轻量级锁在无竞争的情况下使用 CAS 操作去代替使用互斥量。而偏向锁在无竞争的情况下会把整个同步都消除掉**。

偏向锁的“偏”就是偏心的偏，它的意思是会偏向于第一个获得它的线程，如果在接下来的执行中，该锁没有被其他线程获取，那么持有偏向锁的线程就不需要进行同步！

但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。

**② 轻量级锁**

倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)。**轻量级锁不是为了代替重量级锁，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗，因为使用轻量级锁时，不需要申请互斥量。另外，轻量级锁的加锁和解锁都用到了CAS操作。** 关于轻量级锁的加锁和解锁的原理可以查看《深入理解Java虚拟机：JVM高级特性与最佳实践》第二版的13章第三节锁优化。

**轻量级锁能够提升程序同步性能的依据是“对于绝大部分锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用 CAS 操作避免了使用互斥操作的开销。但如果存在锁竞争，除了互斥量开销外，还会额外发生CAS操作，因此在有锁竞争的情况下，轻量级锁比传统的重量级锁更慢！如果锁竞争激烈，那么轻量级将很快膨胀为重量级锁！**

**③ 自旋锁和自适应自旋**

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。

互斥同步对性能最大的影响就是阻塞的实现，因为挂起线程/恢复线程的操作都需要转入内核态中完成（用户态转换到内核态会耗费时间）。

**一般线程持有锁的时间都不是太长，所以仅仅为了这一点时间去挂起线程/恢复线程是得不偿失的。** 所以，虚拟机的开发团队就这样去考虑：“我们能不能让后面来的请求获取锁的线程等待一会而不被挂起呢？看看持有锁的线程是否很快就会释放锁”。**为了让一个线程等待，我们只需要让线程执行一个忙循环（自旋），这项技术就叫做自旋**。

百度百科对自旋锁的解释：

> 何谓自旋锁？它是为实现保护共享资源而提出一种锁机制。其实，自旋锁与互斥锁比较类似，它们都是为了解决对某项资源的互斥使用。无论是互斥锁，还是自旋锁，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。

自旋锁在 JDK1.6 之前其实就已经引入了，不过是默认关闭的，需要通过`--XX:+UseSpinning`参数来开启。JDK1.6及1.6之后，就改为默认开启的了。需要注意的是：自旋等待不能完全替代阻塞，因为它还是要占用处理器时间。如果锁被占用的时间短，那么效果当然就很好了！反之，相反！自旋等待的时间必须要有限度。如果自旋超过了限定次数任然没有获得锁，就应该挂起线程。**自旋次数的默认值是10次，用户可以修改--XX:PreBlockSpin来更改**。

另外,**在 JDK1.6 中引入了自适应的自旋锁。自适应的自旋锁带来的改进就是：自旋的时间不在固定了，而是和前一次同一个锁上的自旋时间以及锁的拥有者的状态来决定，虚拟机变得越来越“聪明”了**。

**④ 锁消除**

锁消除理解起来很简单，它指的就是虚拟机即使编译器在运行时，如果检测到那些共享数据不可能存在竞争，那么就执行锁消除。锁消除可以节省毫无意义的请求锁的时间。

**⑤ 锁粗化**

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小，——直在共享数据的实际作用域才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待线程也能尽快拿到锁。

大部分情况下，上面的原则都是没有问题的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，那么会带来很多不必要的性能消耗。

#### 15. 谈谈 synchronized和ReentrantLock 的区别

**① 两者都是可重入锁**

两者都是可重入锁。“可重入锁”概念是：自己可以再次获取自己的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可锁重入的话，就会造成死锁。同一个线程每次获取锁，锁的计数器都自增1，所以要等到锁的计数器下降为0时才能释放锁。

**② synchronized 依赖于 JVM 而 ReentrantLock 依赖于 API**

synchronized 是依赖于 JVM 实现的，前面我们也讲到了 虚拟机团队在 JDK1.6 为 synchronized 关键字进行了很多优化，但是这些优化都是在虚拟机层面实现的，并没有直接暴露给我们。ReentrantLock 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成），所以我们可以通过查看它的源代码，来看它是如何实现的。

**③ ReentrantLock 比 synchronized 增加了一些高级功能**

相比synchronized，ReentrantLock增加了一些高级功能。主要来说主要有三点：**①等待可中断；②可实现公平锁；③可实现选择性通知（锁可以绑定多个条件）**

- **ReentrantLock提供了一种能够中断等待锁的线程的机制**，通过lock.lockInterruptibly()来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
- **ReentrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。** ReentrantLock默认情况是非公平的，可以通过 ReentrantLock类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
- synchronized关键字与wait()和notify()/notifyAll()方法相结合可以实现等待/通知机制，ReentrantLock类当然也可以实现，但是需要借助于Condition接口与newCondition() 方法。Condition是JDK1.5之后才有的，它具有很好的灵活性，比如可以实现多路通知功能也就是在一个Lock对象中可以创建多个Condition实例（即对象监视器），**线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用notify()/notifyAll()方法进行通知时，被通知的线程是由 JVM 选择的，用ReentrantLock类结合Condition实例可以实现“选择性通知”** ，这个功能非常重要，而且是Condition接口默认提供的。而synchronized关键字就相当于整个Lock对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法的话就会通知所有处于等待状态的线程这样会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。

如果你想使用上述功能，那么选择ReentrantLock是一个不错的选择。

**④ 性能已不是选择标准**

#### 16.volatile ，atomic，synchronized ，threadLocal关键字及区别

##### **16.1 volatile 关键字:**

　　一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰后，就具备了两层语义：保证了不同线程对这个变量进行操作时的可见性和禁止了指令重排序。

　　1、当程序执行到volatile变量的读操作或写操作时，在其之前的操作的更改肯定全部已经进行，且结果对后面的操作可见。其后面的操作肯定还没有进行

　　2、在进行指令优化时，不能将在volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放在其前面执行。

```
volatile关键字的原理和实现机制：
在加入volatile关键字时，会多出一个lock前缀指令。lock前缀指令相当于一个内存屏障，其提供三个功能。
　　1、它会强制将对缓存的修改操作立即写入主内存。
　　2、如果是写操作，它会导致其他CPU中对应的缓存行无效
　　3、它确保指定重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面。即在执行到内存屏障这句指令时，在它前面的操作已经全部完成。
```

##### **16.2 synchronized 关键字和 volatile 关键字的区别:**

- **volatile关键字**是线程同步的**轻量级实现**，所以**volatile性能肯定比synchronized关键字要好**。但是**volatile关键字只能用于变量而synchronized关键字可以修饰方法以及代码块**。synchronized关键字在JavaSE1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁以及其它各种优化之后执行效率有了显著提升，**实际开发中使用 synchronized 关键字的场景还是更多一些**。
- **多线程访问volatile关键字不会发生阻塞，而synchronized关键字可能会发生阻塞**
- **volatile关键字能保证数据的可见性，但不能保证数据的原子性，在非依赖自身的操作的情况下，对变量的改变将对任何线程可见。synchronized关键字两者都能保证**
- **volatile关键字主要用于解决变量在多个线程之间的可见性，而 synchronized关键字解决的是多个线程之间访问资源的同步性。**

##### 16.3 Atomic关键字

要实现原子性,建议使用atomic类的系列对象,支持原子性操作.(注意atomic类只保证本身方法原子性,并不保证多次操作的原子性.除非在方法或对象上加synchronized锁)。

对于原子操作类，Java的concurrent并发包中主要为我们提供了这么几个常用的：AtomicInteger、AtomicLong、AtomicBoolean、AtomicReference<**T**>。

并发包 `java.util.concurrent` 的原子类都存放在`java.util.concurrent.atomic`下,如下图所示。

[![JUC原子类概览](https://camo.githubusercontent.com/b912dfaf3c2985b71a672c7f52c06a6852689f92/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a55432545352538452539462545352541442539302545372542312542422545362541362538322545382541372538382e706e67)](https://camo.githubusercontent.com/b912dfaf3c2985b71a672c7f52c06a6852689f92/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a55432545352538452539462545352541442539302545372542312542422545362541362538322545382541372538382e706e67)

**JUC 包中的原子类是哪4类?**

**基本类型**

使用原子的方式更新基本类型

- AtomicInteger：整形原子类
- AtomicLong：长整型原子类
- AtomicBoolean：布尔型原子类

**数组类型**

使用原子的方式更新数组里的某个元素

- AtomicIntegerArray：整形数组原子类

  **AtomicIntegerArray 类常用方法**

  ```java
  public final int get(int i) //获取 index=i 位置元素的值
  public final int getAndSet(int i, int newValue)//返回 index=i 位置的当前的值，并将其设置为新值：newValue
  public final int getAndIncrement(int i)//获取 index=i 位置元素的值，并让该位置的元素自增
  public final int getAndDecrement(int i) //获取 index=i 位置元素的值，并让该位置的元素自减
  public final int getAndAdd(int delta) //获取 index=i 位置元素的值，并加上预期的值
  boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将 index=i 位置的元素值设置为输入值（update）
  public final void lazySet(int i, int newValue)//最终 将index=i 位置的元素设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
  ```

  **AtomicIntegerArray 常见方法使用**

  ```java
  import java.util.concurrent.atomic.AtomicIntegerArray;
  
  public class AtomicIntegerArrayTest {
  	public static void main(String[] args) {
  		// TODO Auto-generated method stub
  		int temvalue = 0;
  		int[] nums = { 1, 2, 3, 4, 5, 6 };
  		AtomicIntegerArray i = new AtomicIntegerArray(nums);
  		for (int j = 0; j < nums.length; j++) {
  			System.out.println(i.get(j));
  		}
  		temvalue = i.getAndSet(0, 2);
  		System.out.println("temvalue:" + temvalue + ";  i:" + i);
  		temvalue = i.getAndIncrement(0);
  		System.out.println("temvalue:" + temvalue + ";  i:" + i);
  		temvalue = i.getAndAdd(0, 5);
  		System.out.println("temvalue:" + temvalue + ";  i:" + i);
  	}
  }
  ```

- AtomicLongArray：长整形数组原子类

- AtomicReferenceArray：引用类型数组原子类

**引用类型**

- AtomicReference：引用类型原子类

   **AtomicReference 类使用示例**

  ```java
  import java.util.concurrent.atomic.AtomicReference;
  
  public class AtomicReferenceTest {
  
  	public static void main(String[] args) {
  		AtomicReference<Person> ar = new AtomicReference<Person>();
  		Person person = new Person("SnailClimb", 22);
  		ar.set(person);
  		Person updatePerson = new Person("Daisy", 20);
  		ar.compareAndSet(person, updatePerson);
  
  		System.out.println(ar.get().getName());
  		System.out.println(ar.get().getAge());
  	}
  }
  
  class Person {
  	private String name;
  	private int age;
  	public Person(String name, int age) {
  		super();
  		this.name = name;
  		this.age = age;
  	}
  	public String getName() {
  		return name;
  	}
  	public void setName(String name) {
  		this.name = name;
  	}
  	public int getAge() {
  		return age;
  	}
  	public void setAge(int age) {
  		this.age = age;
  	}
  }
  ```

  上述代码首先创建了一个 Person 对象，然后把 Person 对象设置进 AtomicReference 对象中，然后调用 compareAndSet 方法，该方法就是通过通过 CAS 操作设置 ar。如果 ar 的值为 person 的话，则将其设置为 updatePerson。实现原理与 AtomicInteger 类中的 compareAndSet 方法相同。运行上面的代码后的输出结果如下：

- AtomicStampedReference：原子更新引用类型里的字段原子类

  **AtomicStampedReference 类使用示例**

  ```java
  import java.util.concurrent.atomic.AtomicStampedReference;
  
  public class AtomicStampedReferenceDemo {
      public static void main(String[] args) {
          // 实例化、取当前值和 stamp 值
          final Integer initialRef = 0, initialStamp = 0;
          final AtomicStampedReference<Integer> asr = new AtomicStampedReference<>(initialRef, initialStamp);
          System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());
  
          // compare and set
          final Integer newReference = 666, newStamp = 999;
          final boolean casResult = asr.compareAndSet(initialRef, newReference, initialStamp, newStamp);
          System.out.println("currentValue=" + asr.getReference()
                  + ", currentStamp=" + asr.getStamp()
                  + ", casResult=" + casResult);
  
          // 获取当前的值和当前的 stamp 值
          int[] arr = new int[1];
          final Integer currentValue = asr.get(arr);
          final int currentStamp = arr[0];
          System.out.println("currentValue=" + currentValue + ", currentStamp=" + currentStamp);
  
          // 单独设置 stamp 值
          final boolean attemptStampResult = asr.attemptStamp(newReference, 88);
          System.out.println("currentValue=" + asr.getReference()
                  + ", currentStamp=" + asr.getStamp()
                  + ", attemptStampResult=" + attemptStampResult);
  
          // 重新设置当前值和 stamp 值
          asr.set(initialRef, initialStamp);
          System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());
  
          // [不推荐使用，除非搞清楚注释的意思了] weak compare and set
          // 困惑！weakCompareAndSet 这个方法最终还是调用 compareAndSet 方法。[版本: jdk-8u191]
          // 但是注释上写着 "May fail spuriously and does not provide ordering guarantees,
          // so is only rarely an appropriate alternative to compareAndSet."
          // todo 感觉有可能是 jvm 通过方法名在 native 方法里面做了转发
          final boolean wCasResult = asr.weakCompareAndSet(initialRef, newReference, initialStamp, newStamp);
          System.out.println("currentValue=" + asr.getReference()
                  + ", currentStamp=" + asr.getStamp()
                  + ", wCasResult=" + wCasResult);
      }
  }
  ```

- AtomicMarkableReference ：原子更新带有标记位的引用类型

**对象的属性修改类型**

- AtomicIntegerFieldUpdater：原子更新整形字段的更新器
- AtomicLongFieldUpdater：原子更新长整形字段的更新器
- AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

对于原子操作类，最大的特点是在多线程并发操作同一个资源的情况下，使用Lock-Free算法来替代锁，这样开销小、速度快，对于原子操作类是采用原子操作指令实现的，从而可以保证操作的原子性。什么是原子性？比如一个操作i++；实际上这是三个原子操作，先把i的值读取、然后修改(+1)、最后写入给i。所以使用Atomic原子类操作数，比如：i++；那么它会在这步操作都完成情况下才允许其它线程再对它进行操作，而这个实现则是通过Lock-Free+原子操作指令来确定的。而关于Lock-Free算法，则是一种新的策略替代锁来保证资源在并发时的完整性的，Lock-Free的实现有三步：

> 1、循环（for(;;)、while）
> 2、CAS（CompareAndSet）
> 3、回退（return、break）



##### 16.4 Atomic关键字和 volatile 关键字的区别?

java关键字volatile，从表面意思上是说这个变量是易变的，不稳定的，事实上，确实如此，这个关键字的作用就是告诉编译器，凡是被该关键字声明的变量都是易变的、不稳定的。所以不要试图对该变量使用缓存等优化机制，而应当每次都从它的内存地址中去读值。使用volatile标记的变量在读取或写入时不需要使用锁，这将减少产生死锁的概率，使代码保持简洁。请注意，这里只是说每次读取volatile的变量时都要从它的内存地址中读取，**并没有说每次修改完volatile的变量后都要立刻将它的值写回内存**。也就是说volatile只提供了内存可见性，而没有提供原子性，操作互斥提供了操作整体的原子性，同一个变量多个线程间的可见性与多个线程中操作互斥是两件事情，所以说如果用这个关键字做高并发的安全机制的话是不可靠的。

volatile的用法如下：

```
public volatile static int count=0;//在声明的时候带上volatile关键字即可
```

　　什么时候使用volatile关键字？当我们知道了volatile的作用，我们也就知道了它应该用在哪些地方,很显然，**最好是那种只有一个线程修改变量，多个线程读取变量的地方。也就是对内存可见性要求高，而对原子性要求低的地方。**

##### 16.5 能不能给我简单介绍一下 AtomicInteger 类的原理

AtomicInteger 类的部分源码：

```java
    // setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

AtomicInteger 类主要利用 CAS (compare and swap) + volatile 和 native 方法来保证原子操作，从而避免 synchronized 的高开销，执行效率大为提升。

CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。UnSafe 类的 objectFieldOffset() 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

##### 16.6 ThreadLocal及原理

通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。**如果想实现每一个线程都有自己的专属本地变量该如何解决呢？** JDK中提供的`ThreadLocal`类正是为了解决这样的问题。 **ThreadLocal类主要解决的就是让每个线程绑定自己的值，可以将ThreadLocal类形象的比喻成存放数据的盒子，盒子中可以存储每个线程的私有数据。**

**如果你创建了一个ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的本地副本，这也是ThreadLocal变量名的由来。他们可以使用 get（） 和 set（） 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。**

```Java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;

//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```

从上面`Thread`类 源代码可以看出`Thread` 类中有一个 `threadLocals` 和 一个 `inheritableThreadLocals` 变量，它们都是 `ThreadLocalMap` 类型的变量,我们可以把 `ThreadLocalMap` 理解为`ThreadLocal` 类实现的定制化的 `HashMap`。默认情况下这两个变量都是null，只有当前线程调用 `ThreadLocal` 类的 `set`或`get`方法时才创建它们，实际上调用这两个方法的时候，我们调用的是`ThreadLocalMap`类对应的 `get()`、`set() `方法。

`ThreadLocal`类的`set()`方法

```Java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

通过上面这些内容，我们足以通过猜测得出结论：**最终的变量是放在了当前线程的 ThreadLocalMap 中，并不是存在 ThreadLocal 上，ThreadLocal 可以理解为只是ThreadLocalMap的封装，传递了变量值。**

**每个Thread中都具备一个ThreadLocalMap，而ThreadLocalMap可以存储以ThreadLocal为key的键值对。这里解释了为什么每个线程访问同一个ThreadLocal，得到的确是不同的数值。另外，ThreadLocal 是 map结构是为了让每个线程可以关联多个 ThreadLocal变量。**

#####  16.7 ThreadLocal 内存泄露问题

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用,而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候会 key 会被清理掉，而 value 不会被清理掉。这样一来，`ThreadLocalMap` 中就会出现key为null的Entry。假如我们不做任何措施的话，value 永远无法被GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。使用完 `ThreadLocal`方法后 最好手动调用`remove()`方法

```java
      static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

**强引用(StrongReference)**
它类似于必不可少的生活用品,垃圾回收器绝不会回收它。当内存空间不足,Java 虚 拟机宁愿抛出 OutOfMemoryError 错误,使程序异常终止,也不会靠随意回收具有强引用 的对象来解决内存不足问题。强引用是使用最普遍的引用, 以前我们使用的大部分引用实际 上都是强引用。

例如:`Student student = new Student();`
只要此引用存在没要被释放(没有使 student =null),垃圾回收器永远不会回收。只有当这个引用被释放之后,垃圾回收器才可能回收,这也是我们经常所用到的编码形式。

**弱引用(WeakReference)**
如果一个对象只具有弱引用,那就类似于可有可物的生活用品。在垃圾回收器线程扫描 它所管辖的内存区域的过程中,一旦发现了只具有弱引用的对象,不管当前内存空间足够与 否,都会回收它的内存,(不过,由于垃圾回收器是一个优先级很低的线程,因此不一定 会马上发现那些只具有弱引用的对象)如:

`Student student = new Student();//只有student还指向Student就不会被回收`
`WeakRefrence<Student> weakStudent =new WeakRefrence<Student>(student);`
当要获得 weakreference 引用的 student 时,可以使用:weakStudent.get();
如果此方法返回的为空, 那么说明 weakStudent 指向的对象 student 已经被回收了。 例如我们常用的在内部类的 Handler 中使用的 Activity 弱引用,防止内存泄漏。

#### 17.线程池

##### 17.1 为什么要用线程池？

线程池提供了一种限制和管理资源（包括执行一个任务）。 每个线程池还维护一些基本统计信息，例如已完成任务的数量。

- **降低资源消耗:** 通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度:** 当任务到达时，任务可以不需要等到线程创建就能立即执行。
- **提高线程的可管理性:** 线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。
- **解耦作用:**线程的创建于执行完全分开，方便维护。

##### 17.2  如何创建线程池  ?

**方式一：通过构造方法实现** [![ThreadPoolExecutor构造方法](https://camo.githubusercontent.com/c1a87ea139bc0379f5c98484416594843ff29d6d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f546872656164506f6f6c4578656375746f722545362539452538342545392538302541302545362539362542392545362542332539352e706e67)](https://camo.githubusercontent.com/c1a87ea139bc0379f5c98484416594843ff29d6d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f546872656164506f6f6c4578656375746f722545362539452538342545392538302541302545362539362542392545362542332539352e706e67)



**方式二：通过Executor 框架的工具类Executors来实现** 我们可以创建三种类型的ThreadPoolExecutor：

- **FixedThreadPool** ： 该方法返回一个固定线程数量的线程池。该线程池中的线程数量始终不变。当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。
- **SingleThreadExecutor：** 方法返回一个只有一个线程的线程池。若多余一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出的顺序执行队列中的任务。
- **CachedThreadPool：** 该方法返回一个可根据实际情况调整线程数量的线程池。线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。若所有线程均在工作，又有新的任务提交，则会创建新的线程处理任务。所有线程在当前任务执行完毕后，将返回线程池进行复用。

对应Executors工具类中的方法如图所示： [![Executor框架的工具类](https://camo.githubusercontent.com/6cfe663a5033e0f4adcfa148e6c54cdbb97c00bb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4578656375746f722545362541312538362545362539452542362545372539412538342545352542372541352545352538352542372545372542312542422e706e67)](https://camo.githubusercontent.com/6cfe663a5033e0f4adcfa148e6c54cdbb97c00bb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4578656375746f722545362541312538362545362539452542362545372539412538342545352542372541352545352538352542372545372542312542422e706e67)

**《阿里巴巴Java开发手册》中强制线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险**

> Executors 返回线程池对象的弊端如下：
>
> - **FixedThreadPool 和 SingleThreadExecutor** ： 允许请求的队列长度为 Integer.MAX_VALUE ，可能堆积大量的请求，从而导致OOM。
> - **CachedThreadPool 和 ScheduledThreadPool** ： 允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致OOM。



##### 17.3 线程池原理（ThreadPoolExecutor类）

**Executor框架**

Executor框架是java中的线程池实现。Executor是最顶层的接口定义，它的子类和实现主要包括ExecutorService，ScheduledExecutorService，ThreadPoolExecutor，ScheduledThreadPoolExecutor，ForkJoinPool等。其结构如下图所示：

![img](https://upload-images.jianshu.io/upload_images/11963487-4723841eba39749b.png?imageMogr2/auto-orient/strip|imageView2/2/w/538/format/webp)



**Executor**：Executor是一个接口，其只定义了一个execute()方法：`void execute(Runnable command);`，只能提交Runnable形式的任务，不支持提交Callable带有返回值的任务。
**ExecutorService**：ExecutorService在Executor的基础上加入了线程池的生命周期管理，我们可以通过ExecutorService#shutdown或者ExecutorService#shutdownNow方法来关闭我们的线程池。ExecutorService支持提交Callable形式的任务，提交完Callable任务后我们拿到一个Future，它代表一个异步任务执行的结果。关于shutdown和shutdownNow方法我们需要注意的是：这两个方法是非阻塞的，调用后立即返回，不会等待线程池关闭完成。如果我们需要等待线程池处理完成再返回可以使用ExecutorService#awaitTermination来完成。
shutdown方法会等待线程池中已经运行的任何和阻塞队列中等待执行的任务执行完成，而shutdownNow则不会，shutdownNow方法会尝试中断线程池中已经运行的任务，阻塞队列中等待的任务不会再被执行，阻塞队列中等待执行的任务会作为返回值返回。

###### 17.3.1 **线程池状态**

　　在ThreadPoolExecutor中定义了一个volatile变量，另外定义了几个static final变量表示线程池的各个状态：

```Java
volatile int runState;
static final int RUNNING    = 0;
static final int SHUTDOWN   = 1;
static final int STOP       = 2;
static final int TERMINATED = 3;
```

 　　runState表示当前线程池的状态，它是一个volatile变量用来保证线程之间的可见性；

如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

###### 17.3.2 任务的执行

**ThreadPoolExecutor类构造参数：**

```Java
private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小，runState等）的改变都要使用这个锁
private final HashSet<Worker> workers = new HashSet<Worker>(); //用来存放工作集
private volatile long keepAliveTime;    //线程存活时间，这两个参数主要用来控制idle threads的最大空闲时间，超过这个空闲时间空闲线程将被回收。这里有一点需要注意，ThreadPoolExecutor中有一个属性:private volatile boolean allowCoreThreadTimeOut;，这个用来指定是否允许核心线程空闲超时回收，默认为false，即不允许核心线程超时回收，核心线程将一直等待新任务。如果设置这个参数为true，核心线程空闲超时后也可以被回收。
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
private volatile int corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int maximumPoolSize;   //线程池最大能容忍的线程数，当线程池数量已经等于corePoolSize并且阻塞队列也已经满了，则看线程数量是否小于maximumPoolSize：如果小于则创建一个线程来处理请求，否则使用“饱和策略”来拒绝这个请求。对于大于corePoolSize部分的线程，称作这部分线程为“idle threads”，这部分线程会有一个最大空闲时间，如果超过这个空闲时间还没有任务进来则将这些空闲线程回收。
private volatile int poolSize;       //线程池中当前的线程数
private volatile RejectedExecutionHandler handler; //任务拒绝策略 
private volatile ThreadFactory threadFactory; //线程工厂，用来创建线程
private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数
private long completedTaskCount;  //用来记录已经执行完毕的任务个数
```

- 如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；
- 如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务；
- 如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；
- 如果线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

###### 17.3.3 线程池中的线程初始化

　　默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。

　　在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：

```Java
/**初始化一个核心线程**/
public boolean prestartCoreThread() {
    return addIfUnderCorePoolSize(null); //注意传进去的参数是null
}
/**初始化所有核心线程**/
public int prestartAllCoreThreads() {
    int n = 0;
    while (addIfUnderCorePoolSize(null))//注意传进去的参数是null
        ++n;
    return n;
}
```

###### 17.3.4 任务缓存队列及排队策略

　　在前面我们多次提到了任务缓存队列，即workQueue，它用来存放等待执行的任务。

　　workQueue的类型为BlockingQueue<Runnable>，通常可以取下面三种类型：

　　1.ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；

　　2.LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为          Integer.MAX_VALUE；

　　3.synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

###### 17.3.5 任务拒绝策略

当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

```Java
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
```

###### 17.3.6 线程池的关闭

　　ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是shutdown()和shutdownNow()，其中：

- shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务
- shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

###### 17.3.7 线程池容量的动态调整

　　ThreadPoolExecutor提供了动态调整线程池容量大小的方法：setCorePoolSize()和setMaximumPoolSize()，

- setCorePoolSize：设置核心池大小
- setMaximumPoolSize：设置线程池最大能创建的线程数目大小

当上述参数从小变大时，ThreadPoolExecutor进行线程赋值，还可能立即创建新的线程来执行任务。

###### 17.3.8 实现Runnable接口和Callable接口的区别

如果想让线程池执行任务的话需要实现的Runnable接口或Callable接口。 Runnable接口或Callable接口实现类都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。两者的区别在于 Runnable 接口不会返回结果但是 Callable 接口可以返回结果。

**备注：** 工具类`Executors`可以实现`Runnable`对象和`Callable`对象之间的相互转换。（`Executors.callable（Runnable task）`或`Executors.callable（Runnable task，Object resule）`）。

###### 17.3.9. 执行execute()方法和submit()方法的区别是什么呢？

1.**execute() 方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；**

2.**submit() 方法用于提交需要返回值的任务。线程池会返回一个Future类型的对象，通过这个Future对象可以判断任务是否执行成功**，并且可以通过future的get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

##### 18.AQS

###### 18.1 AQS介绍

AQS的全称为（AbstractQueuedSynchronizer），这个类在java.util.concurrent.locks包下面。

[![AQS类](https://camo.githubusercontent.com/7e2bd67b66e3e1764a442b8d96689f64e5521c2c/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4151532545372542312542422e706e67)](https://camo.githubusercontent.com/7e2bd67b66e3e1764a442b8d96689f64e5521c2c/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4151532545372542312542422e706e67)

AQS是一个用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的ReentrantLock，Semaphore，其他的诸如ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。当然，我们自己也能利用AQS非常轻松容易地构造出符合我们自己需求的同步器。

###### 18.2. AQS 原理分析

**AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。**

> CLH(Craig,Landin,and Hagersten)队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装成一个CLH锁队列的一个结点（Node）来实现锁的分配。

看个AQS(AbstractQueuedSynchronizer)原理图：

[![AQS原理图](https://camo.githubusercontent.com/13db51afdebad2dac67a224d422f6f60c9b8d366/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4151532545352538452539462545372539302538362545352539422542452e706e67)](https://camo.githubusercontent.com/13db51afdebad2dac67a224d422f6f60c9b8d366/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4151532545352538452539462545372539302538362545352539422542452e706e67)

AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

```
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

状态信息通过protected类型的getState，setState，compareAndSetState进行操作

```java 
//返回同步状态的当前值
protected final int getState() {  
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
        state = newState;
}
//原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

###### 18.3  AQS 对资源的共享方式

**AQS定义两种资源共享方式**

- **Exclusive**（独占）：只有一个线程能执行，如ReentrantLock。又可分为公平锁和非公平锁：
  - 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
  - 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的
- **Share**（共享）：多个线程可同时执行，如Semaphore/CountDownLatch。Semaphore、CountDownLatch、 CyclicBarrier、ReadWriteLock 我们都会在后面讲到。

ReentrantReadWriteLock 可以看成是组合式，因为ReentrantReadWriteLock也就是读写锁允许多个线程同时对某一资源进行读。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。

###### 18.4 AQS底层使用了模板方法模式

同步器的设计是基于模板方法模式的，如果需要自定义同步器一般的方式是这样（模板方法模式很经典的一个应用）：

1. 使用者继承AbstractQueuedSynchronizer并重写指定的方法。（这些重写方法很简单，无非是对于共享资源state的获取和释放）
2. 将AQS组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

这和我们以往通过实现接口的方式有很大区别，这是模板方法模式很经典的一个运用。

**AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：**

```java 
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

默认情况下，每个方法都抛出 `UnsupportedOperationException`。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS类中的其他方法都是final ，所以无法被其他类使用，只有这几个方法可以被其他类使用。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。

###### 18.5 AQS 组件总结

- **Semaphore(信号量)-允许多个线程同时访问：** synchronized 和 ReentrantLock 都是一次只允许一个线程访问某个资源，Semaphore(信号量)可以指定多个线程同时访问某个资源。

- **CountDownLatch （倒计时器）：** CountDownLatch是一个同步工具类，用来协调多个线程之间的同步。这个工具通常用来控制线程等待，它可以让某一个线程等待直到倒计时结束，再开始执行。

   **CountDownLatch 的三种典型用法**

  ①某一线程在开始运行前等待n个线程执行完毕。将 CountDownLatch 的计数器初始化为n ：`new CountDownLatch(n) `，每当一个任务线程执行完毕，就将计数器减1 `countdownlatch.countDown()`，当计数器的值变为0时，在`CountDownLatch上 await()` 的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

  ②实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 `CountDownLatch` 对象，将其计数器初始化为 1 ：`new CountDownLatch(1) `，多个线程在开始执行任务前首先 `coundownlatch.await()`，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。

  ③死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。

  **CountDownLatch 的使用示例**

  ```java
  /**
   * 
   * @author SnailClimb
   * @date 2018年10月1日
   * @Description: CountDownLatch 使用方法示例
   */
  public class CountDownLatchExample1 {
    // 请求的数量
    private static final int threadCount = 550;
  
    public static void main(String[] args) throws InterruptedException {
      // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
      ExecutorService threadPool = Executors.newFixedThreadPool(300);
      final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
      for (int i = 0; i < threadCount; i++) {
        final int threadnum = i;
        threadPool.execute(() -> {// Lambda 表达式的运用
          try {
            test(threadnum);
          } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          } finally {
            countDownLatch.countDown();// 表示一个请求已经被完成
          }
  
        });
      }
      countDownLatch.await();
      threadPool.shutdown();
      System.out.println("finish");
    }
  
    public static void test(int threadnum) throws InterruptedException {
      Thread.sleep(1000);// 模拟请求的耗时操作
      System.out.println("threadnum:" + threadnum);
      Thread.sleep(1000);// 模拟请求的耗时操作
    }
  }
  ```

  上面的代码中，我们定义了请求的数量为550，当这550个请求被处理完成之后，才会执行`System.out.println("finish");`。

  与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

  其他N个线程必须引用闭锁对象，因为他们需要通知CountDownLatch对象，他们已经完成了各自的任务。这种通知机制是通过 CountDownLatch.countDown()方法来完成的；每调用一次这个方法，在构造函数中初始化的count值就减1。所以当N个线程都调 用了这个方法，count的值等于0，然后主线程就能通过await()方法，恢复执行自己的任务。

  **CountDownLatch 的不足**:

  CountDownLatch是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当CountDownLatch使用完毕后，它不能再次被使用。

- **CyclicBarrier(循环栅栏)：** CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。CyclicBarrier默认的构造方法是 CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await()方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。

  **CyclicBarrier 的应用场景**

  CyclicBarrier 可以用于多线程计算数据，最后合并计算结果的应用场景。比如我们用一个Excel保存了用户所有银行流水，每个Sheet保存一个帐户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水。

  **CyclicBarrier 的使用示例**

  示例1：

  ```java
  /**
   * 
   * @author Snailclimb
   * @date 2018年10月1日
   * @Description: 测试 CyclicBarrier 类中带参数的 await() 方法
   */
  public class CyclicBarrierExample2 {
    // 请求的数量
    private static final int threadCount = 550;
    // 需要同步的线程数量
    private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);
  
    public static void main(String[] args) throws InterruptedException {
      // 创建线程池
      ExecutorService threadPool = Executors.newFixedThreadPool(10);
  
      for (int i = 0; i < threadCount; i++) {
        final int threadNum = i;
        Thread.sleep(1000);
        threadPool.execute(() -> {
          try {
            test(threadNum);
          } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          } catch (BrokenBarrierException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          }
        });
      }
      threadPool.shutdown();
    }
  
    public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
      System.out.println("threadnum:" + threadnum + "is ready");
      try {
        /**等待60秒，保证子线程完全执行结束*/  
        cyclicBarrier.await(60, TimeUnit.SECONDS);
      } catch (Exception e) {
        System.out.println("-----CyclicBarrierException------");
      }
      System.out.println("threadnum:" + threadnum + "is finish");
    }
  
  }
  ```

  运行结果，如下：

  ```java
  threadnum:0is ready
  threadnum:1is ready
  threadnum:2is ready
  threadnum:3is ready
  threadnum:4is ready
  threadnum:4is finish
  threadnum:0is finish
  threadnum:1is finish
  threadnum:2is finish
  threadnum:3is finish
  threadnum:5is ready
  threadnum:6is ready
  threadnum:7is ready
  threadnum:8is ready
  threadnum:9is ready
  threadnum:9is finish
  threadnum:5is finish
  threadnum:8is finish
  threadnum:7is finish
  threadnum:6is finish
  ......
  ```

  可以看到当线程数量也就是请求数量达到我们定义的 5 个的时候， `await`方法之后的方法才被执行。

  另外，CyclicBarrier还提供一个更高级的构造函数`CyclicBarrier(int parties, Runnable barrierAction)`，用于在线程到达屏障时，优先执行`barrierAction`，方便处理更复杂的业务场景。示例代码如下：

  ```java
  /**
   * 
   * @author SnailClimb
   * @date 2018年10月1日
   * @Description: 新建 CyclicBarrier 的时候指定一个 Runnable
   */
  public class CyclicBarrierExample3 {
    // 请求的数量
    private static final int threadCount = 550;
    // 需要同步的线程数量
    private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
      System.out.println("------当线程数达到之后，优先执行------");
    });
  
    public static void main(String[] args) throws InterruptedException {
      // 创建线程池
      ExecutorService threadPool = Executors.newFixedThreadPool(10);
  
      for (int i = 0; i < threadCount; i++) {
        final int threadNum = i;
        Thread.sleep(1000);
        threadPool.execute(() -> {
          try {
            test(threadNum);
          } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          } catch (BrokenBarrierException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
          }
        });
      }
      threadPool.shutdown();
    }
  
    public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
      System.out.println("threadnum:" + threadnum + "is ready");
      cyclicBarrier.await();
      System.out.println("threadnum:" + threadnum + "is finish");
    }
  
  }
  ```

  运行结果，如下：

  ```java
  threadnum:0is ready
  threadnum:1is ready
  threadnum:2is ready
  threadnum:3is ready
  threadnum:4is ready
  ------当线程数达到之后，优先执行------
  threadnum:4is finish
  threadnum:0is finish
  threadnum:2is finish
  threadnum:1is finish
  threadnum:3is finish
  threadnum:5is ready
  threadnum:6is ready
  threadnum:7is ready
  threadnum:8is ready
  threadnum:9is ready
  ------当线程数达到之后，优先执行------
  threadnum:9is finish
  threadnum:5is finish
  threadnum:6is finish
  threadnum:8is finish
  threadnum:7is finish
  ......
  ```

###### 18.6 CyclicBarrier和CountDownLatch的区别

CountDownLatch是计数器，只能使用一次，而CyclicBarrier的计数器提供reset功能，可以多次使用。但是我不那么认为它们之间的区别仅仅就是这么简单的一点。我们来从jdk作者设计的目的来看，javadoc是这么描述它们的：

> CountDownLatch: A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.(CountDownLatch: 一个或者多个线程，等待其他多个线程完成某件事情之后才能执行；) CyclicBarrier : A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.(CyclicBarrier : 多个线程互相等待，直到到达同一个同步点，再继续一起执行。)

对于CountDownLatch来说，重点是“一个线程（多个线程）等待”，而其他的N个线程在完成“某件事情”之后，可以终止，也可以等待。而对于CyclicBarrier，重点是多个线程，在任意一个线程没有完成，所有的线程都必须等待。

CountDownLatch是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而CyclicBarrier更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。

[![CyclicBarrier和CountDownLatch的区别](https://camo.githubusercontent.com/5c19d9e66ffaf3d7193b01948279db9b9b3b98d3/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f4151533333332e706e67)](https://camo.githubusercontent.com/5c19d9e66ffaf3d7193b01948279db9b9b3b98d3/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f4a6176612532302545372541382538422545352542412538462545352539312539382545352542462538352545352541342538372545462542432539412545352542392542362545352538462539312545372539462541352545382541462538362545372542332542422545372542422539462545362538302542422545372542422539332f4151533333332e706e67)

### 二、并发容器总结

JDK提供的这些容器大部分在 `java.util.concurrent` 包中。

- **ConcurrentHashMap:** 线程安全的HashMap
- **CopyOnWriteArrayList:** 线程安全的List，在读多写少的场合性能非常好，远远好于Vector.
- **ConcurrentLinkedQueue:** 高效的并发队列，使用链表实现。可以看做一个线程安全的 LinkedList，这是一个非阻塞队列。
- **BlockingQueue:** 这是一个接口，JDK内部通过链表、数组等方式实现了这个接口。表示阻塞队列，非常适合用于作为数据共享的通道。
- **ConcurrentSkipListMap:** 跳表的实现。这是一个Map，使用跳表的数据结构进行快速查找

#### 1.ConcurrentHashMap

我们知道 HashMap 不是线程安全的，在并发场景下如果要保证一种可行的方式是使用 `Collections.synchronizedMap()`方法来包装我们的 HashMap。但这是通过使用一个全局的锁来同步不同线程间的并发访问，因此会带来不可忽视的性能问题。

所以就有了 HashMap 的线程安全版本—— ConcurrentHashMap 的诞生。在ConcurrentHashMap中，无论是读操作还是写操作都能保证很高的性能：在进行读操作时(几乎)不需要加锁，而在写操作时通过锁分段技术只对所操作的段加锁而不影响客户端对其它段的访问。

关于 ConcurrentHashMap 相关问题，我在 [《这几道Java集合框架面试题几乎必问》](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/Java集合框架常见面试题总结.md) 这篇文章中已经提到过。下面梳理一下关于 ConcurrentHashMap 比较重要的问题：

##### 1.1 ConcurrentHashMap 和 Hashtable 的区别？

ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。

- **底层数据结构：** JDK1.7的 ConcurrentHashMap 底层采用 **分段的数组+链表** 实现，JDK1.8 采用的数据结构跟HashMap1.8的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 **数组+链表** 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；

- **实现线程安全的方式（重要）：** ① **在JDK1.7的时候，ConcurrentHashMap（分段锁）** 对整个桶数组进行了分割分段(Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 **到了 JDK1.8 的时候已经摒弃了Segment的概念，而是直接用 Node 数组+链表+红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6以后 对 synchronized锁做了很多优化）** 整个看起来就像是优化过且线程安全的 HashMap，虽然在JDK1.8中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；② **Hashtable(同一把锁)** :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。[![HashTable全表锁](https://camo.githubusercontent.com/60920968c0e19878ca2a3a153cbe4872f950b571/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f486173685461626c652545352538352541382545382541312541382545392539342538312e706e67)](https://camo.githubusercontent.com/60920968c0e19878ca2a3a153cbe4872f950b571/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f486173685461626c652545352538352541382545382541312541382545392539342538312e706e67)

  **JDK1.7的ConcurrentHashMap：**

  [![JDK1.7的ConcurrentHashMap](https://camo.githubusercontent.com/092aae16c3a38854b4cea8b7e42dc6720df4441f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f436f6e63757272656e74486173684d61702545352538382538362545362541452542352545392539342538312e6a7067)](https://camo.githubusercontent.com/092aae16c3a38854b4cea8b7e42dc6720df4441f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f436f6e63757272656e74486173684d61702545352538382538362545362541452542352545392539342538312e6a7067)

  **JDK1.8的ConcurrentHashMap（TreeBin: 红黑二叉树节点 Node: 链表节点）：**

  [![JDK1.8的ConcurrentHashMap](https://camo.githubusercontent.com/b823c5f2cf18e7e27da70409d2b5e18fed820364/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a444b312e382d436f6e63757272656e74486173684d61702d5374727563747572652e6a7067)](https://camo.githubusercontent.com/b823c5f2cf18e7e27da70409d2b5e18fed820364/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a444b312e382d436f6e63757272656e74486173684d61702d5374727563747572652e6a7067)

##### 1.2 ConcurrentHashMap线程安全的具体实现方式/底层具体实现

**JDK1.7（上面有示意图）**

首先将数据分为一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。

**ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成**。

Segment 实现了 ReentrantLock,所以 Segment 是一种可重入锁，扮演锁的角色。HashEntry 用于存储键值对数据。

```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
}
```

一个 ConcurrentHashMap 里包含一个 Segment 数组。Segment 的结构和HashMap类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个HashEntry数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment的锁。

**JDK1.8 （上面有示意图）**

ConcurrentHashMap取消了Segment分段锁，采用CAS和synchronized来保证并发安全。数据结构跟HashMap1.8的结构类似，数组+链表/红黑二叉树。Java 8在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为O(N)）转换为红黑树（寻址时间复杂度为O(log(N))）

synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash不冲突，就不会产生并发，效率又提升N倍。



#### 2.CopyOnWriteArrayList

##### 2.1 CopyOnWriteArrayList 简介

```java
public class CopyOnWriteArrayList<E>
extends Object
implements List<E>, RandomAccess, Cloneable, Serializable
```

在很多应用场景中，读操作可能会远远大于写操作。由于读操作根本不会修改原有的数据，因此对于每次读取都进行加锁其实是一种资源浪费。我们应该允许多个线程同时访问List的内部数据，毕竟读取操作是安全的。这和我们之前在多线程章节讲过 `ReentrantReadWriteLock` 读写锁的思想非常类似，也就是读读共享、写写互斥、读写互斥、写读互斥。JDK中提供了 `CopyOnWriteArrayList` 类比相比于在读写锁的思想又更进一步。为了将读取的性能发挥到极致，`CopyOnWriteArrayList` 读取是完全不用加锁的，并且更厉害的是：写入也不会阻塞读取操作。只有写入和写入之间需要进行同步等待。这样一来，读操作的性能就会大幅度提升。

##### 2.2 CopyOnWriteArrayList 是如何做到的？

`CopyOnWriteArrayList` 类的所有可变操作（add，set等等）都是通过创建底层数组的新副本来实现的。当 List 需要被修改的时候，我并不修改原有内容，而是对原有数据进行一次复制，将修改的内容写入副本。写完之后，再将修改完的副本替换原来的数据，这样就可以保证写操作不会影响读操作了。

从 `CopyOnWriteArrayList` 的名字就能看出`CopyOnWriteArrayList` 是满足`CopyOnWrite` 的ArrayList，所谓`CopyOnWrite` 也就是说：在计算机，如果你想要对一块内存进行修改时，我们不在原有内存块中进行写操作，而是将内存拷贝一份，在新的内存中进行写操作，写完之后呢，就将指向原来内存指针指向新的内存，原来的内存就可以被回收掉了。

##### 2.3 CopyOnWriteArrayList 读取和写入源码简单分析

###### 2.3.1 CopyOnWriteArrayList 读取操作的实现

读取操作没有任何同步控制和锁操作，理由就是内部数组 array 不会发生修改，只会被另外一个 array 替换，因此可以保证数据安全。

```java
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
    public E get(int index) {
        return get(getArray(), index);
    }
    @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
    final Object[] getArray() {
        return array;
    }
```

###### 2.3.2 CopyOnWriteArrayList 写入操作的实现

CopyOnWriteArrayList 写入操作 add() 方法在添加集合的时候加了锁，保证了同步，避免了多线程写的时候会 copy 出多个副本出来。

```java
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();//加锁
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);//拷贝新数组
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();//释放锁
        }
    }
```



#### 3.ConcurrentLinkedQueue

Java提供的线程安全的 Queue 可以分为**阻塞队列**和**非阻塞队列**，其中阻塞队列的典型例子是 BlockingQueue，非阻塞队列的典型例子是ConcurrentLinkedQueue，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。 **阻塞队列可以通过加锁来实现，非阻塞队列可以通过 CAS 操作实现。**

从名字可以看出，`ConcurrentLinkedQueue`这个队列使用链表作为其数据结构．ConcurrentLinkedQueue 应该算是在高并发环境中性能最好的队列了。它之所有能有很好的性能，是因为其内部复杂的实现。

ConcurrentLinkedQueue 内部代码我们就不分析了，大家知道ConcurrentLinkedQueue 主要使用 CAS 非阻塞算法来实现线程安全就好了。

ConcurrentLinkedQueue 适合在对性能要求相对较高，同时对队列的读写存在多个线程同时进行的场景，即如果对队列加锁的成本较高则适合使用无锁的ConcurrentLinkedQueue来替代。

#### 4.BlockingQueue

面我们己经提到了 ConcurrentLinkedQueue 作为高性能的非阻塞队列。下面我们要讲到的是阻塞队列——BlockingQueue。阻塞队列（BlockingQueue）被广泛使用在“生产者-消费者”问题中，其原因是BlockingQueue提供了可阻塞的插入和移除的方法。当队列容器已满，生产者线程会被阻塞，直到队列未满；当队列容器为空时，消费者线程会被阻塞，直至队列非空时为止。

BlockingQueue 是一个接口，继承自 Queue，所以其实现类也可以作为 Queue 的实现来使用，而 Queue 又继承自 Collection 接口。下面是 BlockingQueue 的相关实现类:

[![BlockingQueue 的实现类](https://camo.githubusercontent.com/e5c9a5d8158376ce80e912cba95ba4919e4377fc/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f35313632323236382e6a7067)](https://camo.githubusercontent.com/e5c9a5d8158376ce80e912cba95ba4919e4377fc/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f35313632323236382e6a7067)

##### 4.1 ArrayBlockingQueue

**ArrayBlockingQueue** 是 BlockingQueue 接口的有界队列实现类，底层采用**数组**来实现。ArrayBlockingQueue一旦创建，容量不能改变。其并发控制采用可重入锁来控制，不管是插入操作还是读取操作，都需要获取到锁才能进行操作。当队列容量满时，尝试将元素放入队列将导致操作阻塞;尝试从一个空队列中取一个元素也会同样阻塞。

ArrayBlockingQueue 默认情况下不能保证线程访问队列的公平性，所谓公平性是指严格按照线程等待的绝对时间顺序，即最先等待的线程能够最先访问到 ArrayBlockingQueue。而非公平性则是指访问 ArrayBlockingQueue 的顺序不是遵守严格的时间顺序，有可能存在，当 ArrayBlockingQueue 可以被访问时，长时间阻塞的线程依然无法访问到 ArrayBlockingQueue。如果保证公平性，通常会降低吞吐量。如果需要获得公平性的 ArrayBlockingQueue，可采用如下代码：

```JAVA
private static ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(10,true);
```

##### 4.2 LinkedBlockingQueue

**LinkedBlockingQueue** 底层基于**单向链表**实现的阻塞队列，可以当做无界队列也可以当做有界队列来使用，同样满足FIFO的特性，与ArrayBlockingQueue 相比起来具有更高的吞吐量，为了防止 LinkedBlockingQueue 容量迅速增，损耗大量内存。通常在创建LinkedBlockingQueue 对象时，会指定其大小，如果未指定，容量等于Integer.MAX_VALUE。

**相关构造方法:**

```java
    /**
     *某种意义上的无界队列
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     *有界队列
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

##### 4.3 PriorityBlockingQueue

**PriorityBlockingQueue** 是一个支持优先级的**无界阻塞队列**。默认情况下元素采用自然顺序进行排序，也可以通过自定义类实现 `compareTo()` 方法来指定元素排序规则，或者初始化时通过构造器参数 `Comparator` 来指定排序规则。

PriorityBlockingQueue 并发控制采用的是 **ReentrantLock**，队列为无界队列（ArrayBlockingQueue 是有界队列，LinkedBlockingQueue 也可以通过在构造函数中传入 capacity 指定队列最大的容量，但是 PriorityBlockingQueue 只能指定初始的队列大小，后面插入元素的时候，**如果空间不够的话会自动扩容**）。

简单地说，它就是 PriorityQueue 的线程安全版本。不可以插入 null 值，同时，插入队列的对象必须是可比较大小的（comparable），否则报 ClassCastException 异常。它的插入操作 put 方法不会 block，因为它是无界队列（take 方法在队列为空的时候会阻塞）。

#### 5. ConcurrentSkipListMap

**为了引出ConcurrentSkipListMap，先带着大家简单理解一下跳表。**

对于一个单链表，即使链表是有序的，如果我们想要在其中查找某个数据，也只能从头到尾遍历链表，这样效率自然就会很低，跳表就不一样了。跳表是一种可以用来快速查找的数据结构，有点类似于平衡树。它们都可以对元素进行快速的查找。但一个重要的区别是：对平衡树的插入和删除往往很可能导致平衡树进行一次全局的调整。而对跳表的插入和删除只需要对整个数据结构的局部进行操作即可。这样带来的好处是：在高并发的情况下，你会需要一个全局锁来保证整个平衡树的线程安全。而对于跳表，你只需要部分锁即可。这样，在高并发环境下，你就可以拥有更好的性能。而就查询的性能而言，跳表的时间复杂度也是 **O(logn)** 所以在并发数据结构中，JDK 使用跳表来实现一个 Map。

跳表的本质是同时维护了多个链表，并且链表是分层的，

[![2级索引跳表](https://camo.githubusercontent.com/11e7cbe718a70a81c42c37a13a257f91ef48dfd7/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f39333636363231372e6a7067)](https://camo.githubusercontent.com/11e7cbe718a70a81c42c37a13a257f91ef48dfd7/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f39333636363231372e6a7067)

最低层的链表维护了跳表内所有的元素，每上面一层链表都是下面一层的子集。

跳表内的所有链表的元素都是排序的。查找时，可以从顶级链表开始找。一旦发现被查找的元素大于当前链表中的取值，就会转入下一层链表继续找。这也就是说在查找过程中，搜索是跳跃式的。如上图所示，在跳表中查找元素18。

[![在跳表中查找元素18](https://camo.githubusercontent.com/111e9bd4936833f774e1a11a2e7bead849ec0ab2/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f33323030353733382e6a7067)](https://camo.githubusercontent.com/111e9bd4936833f774e1a11a2e7bead849ec0ab2/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31322d392f33323030353733382e6a7067)

查找18 的时候原来需要遍历 18 次，现在只需要 7 次即可。针对链表长度比较大的时候，构建索引查找效率的提升就会非常明显。

从上面很容易看出，**跳表是一种利用空间换时间的算法。**

使用跳表实现Map 和使用哈希算法实现Map的另外一个不同之处是：哈希并不会保存元素的顺序，而跳表内所有的元素都是排序的。因此在对跳表进行遍历时，你会得到一个有序的结果。所以，如果你的应用需要有序性，那么跳表就是你不二的选择。JDK 中实现这一数据结构的类是ConcurrentSkipListMap。



### 三、乐观锁与悲观锁

#### 1.悲观锁

总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁（**共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**）。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Java中`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

#### 2.乐观锁

总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。**乐观锁适用于多读的应用类型，这样可以提高吞吐量**，像数据库提供的类似于**write_condition机制**，其实都是提供的乐观锁。在Java中`java.util.concurrent.atomic`包下面的**原子变量类**就是使用了乐观锁的一种实现方式**CAS**实现的。

#### 3.两种锁的使用场景

从上面对两种锁的介绍，我们知道两种锁各有优缺点，不可认为一种好于另一种，像**乐观锁适用于写比较少的情况下（多读场景）**，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果是多写的情况，一般会经常产生冲突，这就会导致上层应用会不断的进行retry，这样反倒是降低了性能，所以**一般多写的场景下用悲观锁就比较合适。**

#### 4.乐观锁常见的两种实现方式

> **乐观锁一般会使用版本号机制或CAS算法实现。**

##### 1. 版本号机制

一般是在数据表中加上一个数据版本号version字段，表示数据被修改的次数，当数据被修改时，version值会加一。当线程A要更新数据值时，在读取数据的同时也会读取version值，在提交更新时，若刚才读取到的version值为当前数据库中的version值相等时才更新，否则重试更新操作，直到更新成功。

**举一个简单的例子：** 假设数据库中帐户信息表中有一个 version 字段，当前值为 1 ；而当前帐户余额字段（ balance ）为 $100 。

1. 操作员 A 此时将其读出（ version=1 ），并从其帐户余额中扣除 $50（ $100-$50 ）。
2. 在操作员 A 操作的过程中，操作员B 也读入此用户信息（ version=1 ），并从其帐户余额中扣除 $20 （ $100-$20 ）。
3. 操作员 A 完成了修改工作，将数据版本号加一（ version=2 ），连同帐户扣除后余额（ balance=$50 ），提交至数据库更新，此时由于提交数据版本大于数据库记录当前版本，数据被更新，数据库记录 version 更新为 2 。
4. 操作员 B 完成了操作，也将版本号加一（ version=2 ）试图向数据库提交数据（ balance=$80 ），但此时比对数据库记录版本时发现，操作员 B 提交的数据版本号为 2 ，数据库记录当前版本也为 2 ，不满足 “ 提交版本必须大于记录当前版本才能执行更新 “ 的乐观锁策略，因此，操作员 B 的提交被驳回。

这样，就避免了操作员 B 用基于 version=1 的旧数据修改的结果覆盖操作员A 的操作结果的可能。

##### 2. CAS算法

即**compare and swap（比较与交换）**，是一种有名的**无锁算法**。无锁编程，即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization）。**CAS算法**涉及到三个操作数

- 需要读写的内存值 V
- 进行比较的值 A
- 拟写入的新值 B

当且仅当 V 的值等于 A时，CAS通过原子方式用新值B来更新V的值，否则不会执行任何操作（比较和替换是一个原子操作）。一般情况下是一个**自旋操作**，即**不断的重试**。



#### 5.乐观锁的缺点

##### 5.1 ABA 问题

如果一个变量V初次读取的时候是A值，并且在准备赋值的时候检查到它仍然是A值，那我们就能说明它的值没有被其他线程修改过了吗？很明显是不能的，因为在这段时间它的值可能被改为其他值，然后又改回A，那CAS操作就会误认为它从来没有被修改过。这个问题被称为CAS操作的 **"ABA"问题。**

JDK 1.5 以后的 `AtomicStampedReference 类`就提供了此种能力，其中的 `compareAndSet 方法`就是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

##### 5.2 循环时间长开销大

**自旋CAS（也就是不成功就一直循环执行直到成功）如果长时间不成功，会给CPU带来非常大的执行开销。** 如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

##### 5.3 只能保证一个共享变量的原子操作

CAS 只对单个共享变量有效，当操作涉及跨多个共享变量时 CAS 无效。但是从 JDK 1.5开始，提供了`AtomicReference类`来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行 CAS 操作.所以我们可以使用锁或者利用`AtomicReference类`把多个共享变量合并成一个共享变量来操作。

#### 6.CAS与synchronized的使用情景

> **简单的来说CAS适用于写比较少的情况下（多读场景，冲突一般较少），synchronized适用于写比较多的情况下（多写场景，冲突一般较多）**

1. 对于资源竞争较少（线程冲突较轻）的情况，使用synchronized同步锁进行线程阻塞和唤醒切换以及用户态内核态间的切换操作额外浪费消耗cpu资源；而CAS基于硬件实现，不需要进入内核，不需要切换线程，操作自旋几率较少，因此可以获得更高的性能。
2. 对于资源竞争严重（线程冲突严重）的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源，效率低于synchronized。

补充： Java并发编程这个领域中synchronized关键字一直都是元老级的角色，很久之前很多人都会称它为 **“重量级锁”** 。但是，在JavaSE 1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的 **偏向锁** 和 **轻量级锁** 以及其它**各种优化**之后变得在某些情况下并不是那么重了。synchronized的底层实现主要依靠 **Lock-Free** 的队列，基本思路是 **自旋后阻塞**，**竞争切换后继续竞争锁**，**稍微牺牲了公平性，但获得了高吞吐量**。在线程冲突较少的情况下，可以获得和CAS类似的性能；而线程冲突严重的情况下，性能远高于CAS。





## 容器

### 一、Java容器基础知识最常见问题

#### 1.说说List,Set,Map三者的区别？

- **List(对付顺序的好帮手)：** List接口存储一组不唯一（可以有多个元素引用相同的对象），有序的对象
- **Set(注重独一无二的性质):** 不允许重复的集合。不会有多个元素引用相同的对象。
- **Map(用Key来搜索的专家):** 使用键值对存储。Map会维护与Key有关联的值。两个Key可以引用相同的对象，但Key不能重复，典型的Key是String类型，但也可以是任何对象。

#### 2.Arraylist 与 LinkedList 区别?

- **1. 是否保证线程安全：** `ArrayList` 和 `LinkedList` 都是不同步的，也就是不保证线程安全；
- **2. 底层数据结构：** `Arraylist` 底层使用的是 **Object 数组**；`LinkedList` 底层使用的是 **双向链表** 数据结构（JDK1.6之前为循环链表，JDK1.7取消了循环。注意双向链表和双向循环链表的区别，下面有介绍到！）
- **3. 插入和删除是否受元素位置的影响：** ① **ArrayList 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。** 比如：执行`add(E e) `方法的时候， `ArrayList` 会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是O(1)。但是如果要在指定位置 i 插入和删除元素的话（`add(int index, E element) `）时间复杂度就为 O(n-i)。因为在进行上述操作的时候集合中第 i 和第 i 个元素之后的(n-i)个元素都要执行向后位/向前移一位的操作。 ② **LinkedList 采用链表存储，所以对于add(�E e)方法的插入，删除元素时间复杂度不受元素位置的影响，近似 O（1），如果是要在指定位置i插入和删除元素的话（(add(int index, E element)） 时间复杂度应为o(n))因为需要新创立一个新的链表，复制前i-1个元素并在第i位加入新的元素，最后附上n-i个元素。**
- **4. 是否支持快速随机访问：** `LinkedList` 不支持高效的随机元素访问，而 `ArrayList` 支持。快速随机访问就是通过元素的序号快速获取元素对象(对应于`get(int index) `方法)。
- **5. 内存空间占用：** ArrayList的空 间浪费主要体现在在list列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间（因为要存放直接后继和直接前驱以及数据）。

##### ***补充内容1:RandomAccess接口***

```Java
public interface RandomAccess {
}
```

查看源码我们发现实际上 `RandomAccess` 接口中什么都没有定义。所以，在我看来 `RandomAccess` 接口不过是一个标识罢了。标识什么？ 标识实现这个接口的类具有随机访问功能。

在 `binarySearch（`）方法中，它要判断传入的list 是否 `RamdomAccess` 的实例，如果是，调用`indexedBinarySearch（）`方法，如果不是，那么调用`iteratorBinarySearch（）`方法

```Java
    public static <T>
    int binarySearch(List<? extends Comparable<? super T>> list, T key) {
        if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
            return Collections.indexedBinarySearch(list, key);
        else
            return Collections.iteratorBinarySearch(list, key);
    }
```

`ArrayList` 实现了 `RandomAccess` 接口， 而 `LinkedList` 没有实现。为什么呢？我觉得还是和底层数据结构有关！`ArrayList` 底层是数组，而 `LinkedList` 底层是链表。数组天然支持随机访问，时间复杂度为 O（1），所以称为快速随机访问。链表需要遍历到特定位置才能访问特定位置的元素，时间复杂度为 O（n），所以不支持快速随机访问。`ArrayList` 实现了 `RandomAccess` 接口，就表明了他具有快速随机访问功能。 `RandomAccess` 接口只是标识，并不是说 `ArrayList` 实现 `RandomAccess` 接口才具有快速随机访问功能的！

**下面再总结一下 list 的遍历方式选择：**

- 实现了 `RandomAccess` 接口的list，优先选择普通 for 循环 ，其次 foreach,
- 未实现 `RandomAccess`接口的list，优先选择iterator遍历（foreach遍历底层也是通过iterator实现的,），大size的数据，千万不要使用普通for循环

##### ***补充内容2:双向链表和双向循环链表：***

**双向链表：** 包含两个指针，一个prev指向前一个节点，一个next指向后一个节点。

**双向链表(双链表)是链表的一种。和单链表一样，双链表也是由节点组成，它的每个数据结点中都有两个指针，分别指向直接后继和直接前驱。**

![img](https://images2017.cnblogs.com/blog/922236/201712/922236-20171220110011240-853997173.png)

**双向循环链表：**从双向链表中的任意一个结点开始，都可以很方便地访问它的前驱结点和后继结点。一般我们都构造双向循环链表。

双链表的示意图如下：

![img](https://images2015.cnblogs.com/blog/632293/201703/632293-20170306150003297-951612143.png)

表头为空，表头的后继节点为"节点10"(数据为10的节点)；"节点10"的后继节点是"节点20"(数据为10的节点)，"节点20"的前继节点是"节点10"；"节点20"的后继节点是"节点30"，"节点30"的前继节点是"节点20"；...；末尾节点的后继节点是表头。

**双链表删除节点**

**![img](https://images2015.cnblogs.com/blog/632293/201703/632293-20170306150039609-1719195275.png)**

删除"节点30"
**删除之前**："节点20"的后继节点为"节点30"，"节点30" 的前继节点为"节点20"。"节点30"的后继节点为"节点40"，"节点40" 的前继节点为"节点30"。
**删除之后**："节点20"的后继节点为"节点40"，"节点40" 的前继节点为"节点20"。

 **双链表添加节点**

 ![img](https://images2015.cnblogs.com/blog/632293/201703/632293-20170306150105453-2068434988.png)

在"节点10"与"节点20"之间添加"节点15"
**添加之前**："节点10"的后继节点为"节点20"，"节点20" 的前继节点为"节点10"。
**添加之后**："节点10"的后继节点为"节点15"，"节点15" 的前继节点为"节点10"。"节点15"的后继节点为"节点20"，"节点20" 的前继节点为"节点15"。

#### 3.ArrayList 与 Vector 区别呢?为什么要用Arraylist取代Vector呢？

Vector是同步的。这个类中的一些方法保证了Vector中的对象是线程安全的。而ArrayList则是异步的，因此ArrayList中的对象并不是线程安全的。因为同步的要求会影响执行的效率，所以如果你不需要线程安全的集合那么使用ArrayList是一个很好的选择，这样可以避免由于同步带来的不必要的性能开销。

数据增长从内部实现机制来讲ArrayList和Vector都是使用数组(Array)来控制集合中的对象。当你向这两种类型中增加元素的时候，如果元素的数目超出了内部数组目前的长度它们都需要扩展内部数组的长度，Vector缺省情况下自动增长原来一倍的数组长度，ArrayList是原来的50%,所以最后你获得的这个集合所占的空间总是比你实际需要的要大。所以如果你要在集合中保存大量的数据那么使用Vector有一些优势，因为你可以通过设置集合的初始化大小来避免不必要的资源开销。

使用模式在ArrayList和Vector中，从一个指定的位置（通过索引）查找数据或是在集合的末尾增加、移除一个元素所花费的时间是一样的，这个时间我们用O(1)表示。但是，如果在集合的其他位置增加或移除元素那么花费的时间会呈线形增长：O(n-i)，其中n代表集合中元素的个数，i代表元素增加或移除元素的索引位置。

#### 4.HashMap 和 Hashtable 的区别?

1. **线程是否安全：** HashMap 是非线程安全的，HashTable 是线程安全的；HashTable 内部的方法基本都经过`synchronized` 修饰。（如果你要保证线程安全的话就使用 ConcurrentHashMap 吧！）；
2. **效率：** 因为线程安全的问题，HashMap 要比 HashTable 效率高一点。另外，HashTable 基本被淘汰，不要在代码中使用它；
3. **对Null key 和Null value的支持：** HashMap 中，null 可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为 null。但是在 HashTable 中 put 进的键值只要有一个 null，直接抛出 NullPointerException。
4. **初始容量大小和每次扩充容量大小的不同 ：** ①创建时如果不指定容量初始值，Hashtable 默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap 默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。②创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 HashMap 会将其扩充为2的幂次方大小（HashMap 中的`tableSizeFor()`方法保证，下面给出了源代码）。也就是说 HashMap 总是使用2的幂作为哈希表的大小,后面会介绍到为什么是2的幂次方。
5. **底层数据结构：** JDK1.8 以后的 HashMap 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

**HasMap 中带有初始容量的构造函数：**

```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
     public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

下面这个方法保证了 HashMap 总是使用2的幂作为哈希表的大小。

```java 
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

#### 5.HashMap 和 HashSet区别?

如果你看过 HashSet 源码的话就应该知道：HashSet 底层就是基于 HashMap 实现的。（HashSet 的源码非常非常少，因为除了 `clone() `、`writeObject()`、`readObject()`是 HashSet 自己不得不实现之外，其他方法都是直接调用 HashMap 中的方法。

| HashMap                          | HashSet                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| 实现了Map接口                    | 实现Set接口                                                  |
| 存储键值对                       | 仅存储对象                                                   |
| 调用 `put（）`向map中添加元素    | 调用 `add（）`方法向Set中添加元素                            |
| HashMap使用键（Key）计算Hashcode | HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性 |

#### 6.HashSet如何检查重复?

当你把对象加入HashSet时，HashSet会先计算对象的hashcode值来判断对象加入的位置，同时也会与其他加入的对象的hashcode值作比较，如果没有相符的hashcode，HashSet会假设对象没有重复出现。但是如果发现有相同hashcode值的对象，这时会调用equals（）方法来检查hashcode相等的对象是否真的相同。如果两者相同，HashSet就不会让加入操作成功。

***hashCode（）与equals（）的相关规定：***

1. 如果两个对象相等，则hashcode一定也是相同的
2. 两个对象相等,对两个equals方法返回true
3. 两个对象有相同的hashcode值，它们也不一定是相等的
4. 综上，equals方法被覆盖过，则hashCode方法也必须被覆盖
5. hashCode()的默认行为是对堆上的对象产生独特值。如果没有重写hashCode()，则该class的两个对象无论如何都不会相等（即使这两个对象指向相同的数据）。

***==与equals的区别?***

| ==                                         |                          equals                          |
| ------------------------------------------ | :------------------------------------------------------: |
| 判断两个变量或实例是不是指向同一个内存空间 | equals是判断两个变量或实例所指向的内存空间的值是不是相同 |
| ==是指对内存地址进行比较                   |             equals()是对字符串的内容进行比较             |
| ==指引用是否相同                           |                 equals()指的是值是否相同                 |

#### 7.HashMap的底层实现?

**JDK1.8之前:**

JDK1.8 之前 HashMap 底层是 **数组和链表** 结合在一起使用也就是 **链表散列**。HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值，然后通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

所谓扰动函数指的就是 HashMap 的 hash 方法。使用 hash 方法也就是扰动函数是为了防止一些实现比较差的 hashCode() 方法 换句话说使用扰动函数之后可以减少碰撞。

JDK 1.8 的 hash方法 相比于 JDK 1.7 hash 方法更加简化，但是原理不变。

```java
    static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^ ：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

对比一下 JDK1.7的 HashMap 的 hash 方法源码

```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。

所谓 **“拉链法”** 就是：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

[![jdk1.8之前的内部结构](https://camo.githubusercontent.com/eec1c575aa5ff57906dd9c9130ec7a82e212c96a/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f332f32302f313632343064626363333033643837323f773d33343826683d34323726663d706e6726733d3130393931)](https://camo.githubusercontent.com/eec1c575aa5ff57906dd9c9130ec7a82e212c96a/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f332f32302f313632343064626363333033643837323f773d33343826683d34323726663d706e6726733d3130393931)



**JDK1.8之后**:

相比于之前的版本， JDK1.8之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。

[![JDK1.8之后的HashMap底层数据结构](https://camo.githubusercontent.com/20de7e465cac279842851258ec4d1ec1c4d3d7d1/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32322f36373233333736342e6a7067)](https://camo.githubusercontent.com/20de7e465cac279842851258ec4d1ec1c4d3d7d1/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32322f36373233333736342e6a7067)

> TreeMap、TreeSet以及JDK1.8之后的HashMap底层都用到了红黑树。红黑树就是为了解决二叉查找树的缺陷，因为二叉查找树在某些情况下会退化成一个线性结构。



#### 8.HashMap 的长度为什么是2的幂次方？

为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash 值的范围值-2147483648（-2^31）到2147483647（2^31-1），前后加起来大概40亿（2^32）的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ `(n - 1) & hash`”。（n代表数组长度）。这也就解释了 HashMap 的长度为什么是2的幂次方。

**这个算法应该如何设计呢？**

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：**“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。”** 并且 **采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。**

#### 9.HashMap 多线程操作导致死循环问题？

主要原因在于 并发下的Rehash 会造成元素之间会形成一个循环链表。不过，jdk 1.8 后解决了这个问题，但是还是不建议在多线程下使用 HashMap,因为多线程下使用 HashMap 还是会存在其他问题比如数据丢失。并发环境下推荐使用 ConcurrentHashMap 。

#### 10.comparable 和 Comparator的区别

- comparable接口实际上是出自java.lang包 它有一个 `compareTo(Object obj)`方法用来排序
- comparator接口实际上是出自 java.util 包它有一个`compare(Object obj1, Object obj2)`方法用来排序

一般我们需要对一个集合使用自定义排序时，我们就要重写`compareTo()`方法或`compare()`方法，当我们需要对某一个集合实现两种排序方式，比如一个song对象中的歌名和歌手名分别采用一种排序方法的话，我们可以重写`compareTo()`方法和使用自制的Comparator方法或者以两个Comparator来实现歌名排序和歌星名排序，第二种代表我们只能使用两个参数版的 `Collections.sort()`.

**Comparator定制排序**

```java
        ArrayList<Integer> arrayList = new ArrayList<Integer>();
        arrayList.add(-1);
        arrayList.add(3);
        arrayList.add(3);
        arrayList.add(-5);
        arrayList.add(7);
        arrayList.add(4);
        arrayList.add(-9);
        arrayList.add(-7);
        System.out.println("原始数组:");
        System.out.println(arrayList);
        // void reverse(List list)：反转
        Collections.reverse(arrayList);
        System.out.println("Collections.reverse(arrayList):");
        System.out.println(arrayList);

        // void sort(List list),按自然排序的升序排序
        Collections.sort(arrayList);
        System.out.println("Collections.sort(arrayList):");
        System.out.println(arrayList);
        // 定制排序的用法
        Collections.sort(arrayList, new Comparator<Integer>() {

            @Override
            public int compare(Integer o1, Integer o2) {
                return o2.compareTo(o1);
            }
        });
        System.out.println("定制排序后：");
        System.out.println(arrayList);
```

Output:

```
原始数组:
[-1, 3, 3, -5, 7, 4, -9, -7]
Collections.reverse(arrayList):
[-7, -9, 4, 7, -5, 3, 3, -1]
Collections.sort(arrayList):
[-9, -7, -5, -1, 3, 3, 4, 7]
定制排序后：
[7, 4, 3, 3, -1, -5, -7, -9]
```

**重写compareTo方法实现按年龄来排序**

```java
// person对象没有实现Comparable接口，所以必须实现，这样才不会出错，才可以使treemap中的数据按顺序排列
// 前面一个例子的String类已经默认实现了Comparable接口，详细可以查看String类的API文档，另外其他
// 像Integer类等都已经实现了Comparable接口，所以不需要另外实现了
public  class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    /**
     * TODO重写compareTo方法实现按年龄来排序
     */
    @Override
    public int compareTo(Person o) {
        // TODO Auto-generated method stub
        if (this.age > o.getAge()) {
            return 1;
        } else if (this.age < o.getAge()) {
            return -1;
        }
        return age;
    }
}
    public static void main(String[] args) {
        TreeMap<Person, String> pdata = new TreeMap<Person, String>();
        pdata.put(new Person("张三", 30), "zhangsan");
        pdata.put(new Person("李四", 20), "lisi");
        pdata.put(new Person("王五", 10), "wangwu");
        pdata.put(new Person("小红", 5), "xiaohong");
        // 得到key的值的同时得到key所对应的值
        Set<Person> keys = pdata.keySet();
        for (Person key : keys) {
            System.out.println(key.getAge() + "-" + key.getName());

        }
    }
```

Output：

```java
5-小红
10-王五
20-李四
30-张三
```



### 二、源码学习

#### ArrayList源码：



#### LinkedList源码：

#### HashMap源码：























## JVM

### 一、Java内存区域

#### 1.运行时数据区域

Java 虚拟机在执行 Java 程序的过程中会把它管理的内存划分成若干个不同的数据区域。JDK. 1.8 和之前的版本略有不同，下面会介绍到。

**JDK 1.8 之前：**

[![img](https://camo.githubusercontent.com/a66819fd82c6adfa69b368edf3c52b1fa9cdc89d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d332f4a564de8bf90e8a18ce697b6e695b0e68daee58cbae59f9f2e706e67)](https://camo.githubusercontent.com/a66819fd82c6adfa69b368edf3c52b1fa9cdc89d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d332f4a564de8bf90e8a18ce697b6e695b0e68daee58cbae59f9f2e706e67)

**JDK 1.8 ：**

[![img](https://camo.githubusercontent.com/0bcc6c01a919b175827f0d5540aeec115df6c001/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d334a617661e8bf90e8a18ce697b6e695b0e68daee58cbae59f9f4a444b312e382e706e67)](https://camo.githubusercontent.com/0bcc6c01a919b175827f0d5540aeec115df6c001/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d334a617661e8bf90e8a18ce697b6e695b0e68daee58cbae59f9f4a444b312e382e706e67)

**线程私有的：**

- 程序计数器
- 虚拟机栈
- 本地方法栈

**线程共享的：**

- 堆
- 方法区
- 直接内存 (非运行时数据区的一部分)

##### 1.1 程序计数器

程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。**字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成。**

另外，**为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。**

**从上面的介绍中我们知道程序计数器主要有两个作用：**

1. 字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
2. 在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。

**注意：程序计数器是唯一一个不会出现 OutOfMemoryError 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。**

##### 1.2 Java 虚拟机栈

与程序计数器一样，Java 虚拟机栈也是线程私有的，它的生命周期和线程相同，描述的是 Java 方法执行的内存模型，每次方法调用的数据都是通过栈传递的。

**Java 内存可以粗糙的区分为堆内存（Heap）和栈内存 (Stack),其中栈就是现在说的虚拟机栈，或者说是虚拟机栈中局部变量表部分。** （实际上，Java 虚拟机栈是由一个个栈帧组成，而每个栈帧中都拥有：**局部变量表、操作数栈、动态链接、方法出口信息。）**

**局部变量表主要存放了编译器可知的各种数据类型**（boolean、byte、char、short、int、float、long、double）、**对象引用**（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。

**Java 虚拟机栈会出现两种异常：StackOverFlowError 和 OutOfMemoryError。**

- **StackOverFlowError：** 若 Java 虚拟机栈的内存大小不允许动态扩展，那么当线程请求栈的深度超过当前 Java 虚拟机栈的最大深度的时候，就抛出 StackOverFlowError 异常。
- **OutOfMemoryError：** 若 Java 虚拟机栈的内存大小允许动态扩展，且当线程请求栈时内存用完了，无法再动态扩展了，此时抛出 OutOfMemoryError 异常。

Java 虚拟机栈也是线程私有的，每个线程都有各自的 Java 虚拟机栈，而且随着线程的创建而创建，随着线程的死亡而死亡。

**扩展：那么方法/函数如何调用？**

Java 栈可用类比数据结构中栈，Java 栈中保存的主要内容是栈帧，每一次函数调用都会有一个对应的栈帧被压入 Java 栈，每一个函数调用结束后，都会有一个栈帧被弹出。

Java 方法有两种返回方式：

1. return 语句。
2. 抛出异常。

不管哪种返回方式都会导致栈帧被弹出。

##### 1.3 本地方方法栈

和虚拟机栈所发挥的作用非常相似，区别是： **虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

本地方法被执行的时候，在本地方法栈也会创建一个栈帧，用于存放该本地方法的局部变量表、操作数栈、动态链接、出口信息。

方法执行完毕后相应的栈帧也会出栈并释放内存空间，也会出现 StackOverFlowError 和 OutOfMemoryError 两种异常。

##### 1.4 堆

Java 虚拟机所管理的内存中最大的一块，Java 堆是所有线程共享的一块内存区域，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

Java 堆是垃圾收集器管理的主要区域，因此也被称作**GC 堆（Garbage Collected Heap）**.从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：新生代和老年代：再细致一点有：Eden 空间、From Survivor、To Survivor 空间等。**进一步划分的目的是更好地回收内存，或者更快地分配内存。**

[![img](https://camo.githubusercontent.com/4012482f49926b35d8557b63952ee605fd259f62/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33e5a086e7bb93e69e842e706e67)](https://camo.githubusercontent.com/4012482f49926b35d8557b63952ee605fd259f62/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33e5a086e7bb93e69e842e706e67)

上图所示的 eden 区、s0 区、s1 区都属于新生代，tentired 区属于老年代。大部分情况，对象都会首先在 Eden 区域分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 s0 或者 s1，并且对象的年龄还会加 1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。

##### 1.5 方法区

方法区与 Java 堆一样，是各个线程共享的内存区域，**它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**。虽然 Java 虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 **Non-Heap（非堆）**，目的应该是与 Java 堆区分开来。

方法区也被称为永久代。很多人都会分不清方法区和永久代的关系，为此我也查阅了文献。

###### 1.5.1 方法区和永久代的关系

> 《Java 虚拟机规范》只是规定了有方法区这么个概念和它的作用，并没有规定如何去实现它。那么，在不同的 JVM 上方法区的实现肯定是不同的了。 **方法区和永久代的关系很像 Java 中接口和类的关系，类实现了接口，而永久代就是 HotSpot 虚拟机对虚拟机规范中方法区的一种实现方式。** 也就是说，永久代是 HotSpot 的概念，方法区是 Java 虚拟机规范中的定义，是一种规范，而永久代是一种实现，一个是标准一个是实现，其他的虚拟机实现并没有永久代这一说法。

###### 1.5.2 常用参数

JDK 1.8 之前永久代还没被彻底移除的时候通常通过下面这些参数来调节方法区大小

```java 
-XX:PermSize=N //方法区 (永久代) 初始大小
-XX:MaxPermSize=N //方法区 (永久代) 最大大小,超过这个值将会抛出 OutOfMemoryError 异常:java.lang.OutOfMemoryError: PermGen
```

相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入方法区后就“永久存在”了。

JDK 1.8 的时候，方法区（HotSpot 的永久代）被彻底移除了（JDK1.7 就已经开始了），取而代之是元空间，元空间使用的是直接内存。

下面是一些常用参数：

```java
-XX:MetaspaceSize=N //设置 Metaspace 的初始（和最小大小）
-XX:MaxMetaspaceSize=N //设置 Metaspace 的最大大小
```

与永久代很大的不同就是，如果不指定大小的话，随着更多类的创建，虚拟机会耗尽所有可用的系统内存。

###### 1.5.3 为什么要将永久代 (PermGen) 替换为元空间 (MetaSpace) 呢?

整个永久代有一个 JVM 本身设置固定大小上限，无法进行调整，而元空间使用的是直接内存，受本机可用内存的限制，并且永远不会得到 java.lang.OutOfMemoryError。你可以使用 `-XX：MaxMetaspaceSize` 标志设置最大元空间大小，默认值为 unlimited，这意味着它只受系统内存的限制。`-XX：MetaspaceSize` 调整标志定义元空间的初始大小如果未指定此标志，则 Metaspace 将根据运行时的应用程序需求动态地重新调整大小。

当然这只是其中一个原因，还有很多底层的原因，这里就不提了。

##### 1.6 运行时常量池

运行时常量池是方法区的一部分。**Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池信息（用于存放编译期生成的各种字面量和符号引用）**

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

**JDK1.7 及之后版本的 JVM 已经将运行时常量池从方法区中移了出来，在 Java 堆（Heap）中开辟了一块区域存放运行时常量池。**

[![img](https://camo.githubusercontent.com/17620721a9f326a235aeec8956949cec03f3f125/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d392d31342f32363033383433332e6a7067)](https://camo.githubusercontent.com/17620721a9f326a235aeec8956949cec03f3f125/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d392d31342f32363033383433332e6a7067)

##### 1.7 直接内存

**直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 异常出现。**

JDK1.4 中新加入的 **NIO(New Input/Output) 类**，引入了一种基于**通道（Channel）** 与**缓存区（Buffer）** 的 I/O 方式，它可以直接使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为**避免了在 Java 堆和 Native 堆之间来回复制数据**。

本机直接内存的分配不会受到 Java 堆的限制，但是，既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。



#### 2.HotSpot 虚拟机对象探秘

##### 2.1 对象的创建

下图便是 Java 对象的创建过程，我建议最好是能默写出来，并且要掌握每一步在做什么。 [![Java创建对象的过程](https://camo.githubusercontent.com/8adbddf019488872c5da890c3bee263db22150fe/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a6176612545352538382539422545352542422542412545352541462542392545382542312541312545372539412538342545382542462538372545372541382538422e706e67)](https://camo.githubusercontent.com/8adbddf019488872c5da890c3bee263db22150fe/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f4a6176612545352538382539422545352542422542412545352541462542392545382542312541312545372539412538342545382542462538372545372541382538422e706e67)

###### Step1:类加载检查

虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

###### Step2:分配内存

在**类加载检查**通过后，接下来虚拟机将为新生对象**分配内存**。对象所需的内存大小在类加载完成后便可确定，为对象分配空间的任务等同于把一块确定大小的内存从 Java 堆中划分出来。**分配方式**有 **“指针碰撞”** 和 **“空闲列表”** 两种，**选择那种分配方式由 Java 堆是否规整决定，而 Java 堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定**。

**内存分配的两种方式：（补充内容，需要掌握）**

选择以上两种方式中的哪一种，取决于 Java 堆内存是否规整。而 Java 堆内存是否规整，取决于 GC 收集器的算法是"标记-清除"，还是"标记-整理"（也称作"标记-压缩"），值得注意的是，复制算法内存也是规整的

|          | 指针碰撞                                                     | 空闲列表                                                     |
| :------: | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 适用场合 | 堆内存规整（即没有内存碎片）                                 | 堆内存不规整的情况                                           |
|   原理   | 用过的内存全部整合到一边，没用过的内存放在另一边，中间有个分界值指针，只需要向着没用过的内存方向将该指针移动对象内存大小的位置即可 | 虚拟机会维护一个列表，该列表中会记录哪些内存块是可用的，在分配的时候，找一块内存足够大的内存块来分配给对象实例，最后更新列表记录 |
| GC收集器 | Serial、ParNew                                               | CMS                                                          |

**内存分配并发问题（补充内容，需要掌握）**

在创建对象的时候有一个很重要的问题，就是线程安全，因为在实际开发过程中，创建对象是很频繁的事情，作为虚拟机来说，必须要保证线程是安全的，通常来讲，虚拟机采用两种方式来保证线程安全：

- **CAS+失败重试：** CAS 是乐观锁的一种实现方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。**虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。**
- **TLAB：** 为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS 进行内存分配

###### Step3:初始化零值

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

###### Step4:设置对象头

初始化零值完成之后，**虚拟机要对对象进行必要的设置**，例如这个对象是那个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的 GC 分代年龄等信息。 **这些信息存放在对象头中。** 另外，根据虚拟机当前运行状态的不同，如是否启用偏向锁等，对象头会有不同的设置方式。

###### Step5:执行 init 方法

在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，但从 Java 程序的视角来看，对象创建才刚开始，`<init>` 方法还没有执行，所有的字段都还为零。所以一般来说，执行 new 指令之后会接着执行 `<init>` 方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来。

##### 2.2 对象的内存布局

在 Hotspot 虚拟机中，对象在内存中的布局可以分为 3 块区域：**对象头**、**实例数据**和**对齐填充**。

**Hotspot 虚拟机的对象头包括两部分信息**，**第一部分用于存储对象自身的自身运行时数据**（哈希码、GC 分代年龄、锁状态标志等等），**另一部分是类型指针**，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是那个类的实例。

**实例数据部分是对象真正存储的有效信息**，也是在程序中所定义的各种类型的字段内容。

**对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。** 因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说就是对象的大小必须是 8 字节的整数倍。而对象头部分正好是 8 字节的倍数（1 倍或 2 倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

##### 2.3 对象的访问定位

建立对象就是为了使用对象，我们的 Java 程序通过栈上的 reference 数据来操作堆上的具体对象。对象的访问方式由虚拟机实现而定，目前主流的访问方式有**①使用句柄**和**②直接指针**两种：

1. **句柄：** 如果使用句柄的话，那么 Java 堆中将会划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息；

[![对象的访问定位-使用句柄](https://camo.githubusercontent.com/04c82b46121149c8cc9c3b81e18967a5ce06353f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541462542392545382542312541312545372539412538342545382541452542462545392539372541452545352541452539412545342542442538442d2545342542442542462545372539342541382545352538462541352545362539462538342e706e67)](https://camo.githubusercontent.com/04c82b46121149c8cc9c3b81e18967a5ce06353f/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541462542392545382542312541312545372539412538342545382541452542462545392539372541452545352541452539412545342542442538442d2545342542442542462545372539342541382545352538462541352545362539462538342e706e67)

1. **直接指针：** 如果使用直接指针访问，那么 Java 堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而 reference 中存储的直接就是对象的地址。（**Hotspot采用的就是直接指针方式**）

[![对象的访问定位-直接指针](https://camo.githubusercontent.com/0ae309b058b45ee14004cd001e334355231b2246/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541462542392545382542312541312545372539412538342545382541452542462545392539372541452545352541452539412545342542442538442d2545372539422542342545362538452541352545362538432538372545392539322538382e706e67)](https://camo.githubusercontent.com/0ae309b058b45ee14004cd001e334355231b2246/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541462542392545382542312541312545372539412538342545382541452542462545392539372541452545352541452539412545342542442538442d2545372539422542342545362538452541352545362538432538372545392539322538382e706e67)

**这两种对象访问方式各有优势。使用句柄来访问的最大好处是 reference 中存储的是稳定的句柄地址，在对象被移动时只会改变句柄中的实例数据指针，而 reference 本身不需要修改。使用直接指针访问方式最大的好处就是速度快，它节省了一次指针定位的时间开销。**



#### 3.重点补充内容

##### 3.1 String 类和常量池

**String 对象的两种创建方式：**

```java
String str1 = "abcd";//先检查字符串常量池中有没有"abcd"，如果字符串常量池中没有，则创建一个，然后 str1 指向字符串常量池中的对象，如果有，则直接将 str1 指向"abcd""；
String str2 = new String("abcd");//堆中创建一个新的对象
String str3 = new String("abcd");//堆中创建一个新的对象
System.out.println(str1==str2);//false
System.out.println(str2==str3);//false
```

这两种不同的创建方法是有差别的。

- 第一种方式是在常量池中拿对象；
- 第二种方式是直接在堆内存空间创建一个新的对象。

记住一点：**只要使用 new 方法，便需要创建新的对象。**

再给大家一个图应该更容易理解，图片来源：https://www.journaldev.com/797/what-is-java-string-pool：

[![String-Pool-Java](https://camo.githubusercontent.com/48189454746b5979fd465b32b996d222619ac1dc/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33537472696e672d506f6f6c2d4a617661312d343530783234392e706e67)](https://camo.githubusercontent.com/48189454746b5979fd465b32b996d222619ac1dc/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33537472696e672d506f6f6c2d4a617661312d343530783234392e706e67)

**String 类型的常量池比较特殊。它的主要使用方法有两种：**

- 直接使用双引号声明出来的 String 对象会直接存储在常量池中。
- 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern 方法。**String.intern() 是一个 Native 方法，它的作用是：如果运行时常量池中已经包含一个等于此 String 对象内容的字符串，则返回常量池中该字符串的引用**；**如果没有，JDK1.7之前（不包含1.7）的处理方式是在常量池中创建与此 String 内容相同的字符串，并返回常量池中创建的字符串的引用，JDK1.7以及之后的处理方式是在常量池中记录此字符串的引用，并返回该引用。**

```java
	      String s1 = new String("计算机");
	      String s2 = s1.intern();
	      String s3 = "计算机";
	      System.out.println(s2);//计算机
	      System.out.println(s1 == s2);//false，因为一个是堆内存中的 String 对象一个是常量池中的 										String 对象
	      System.out.println(s3 == s2);//true，因为两个都是常量池中的 String 对象
```

**字符串拼接:**

```java
		  String str1 = "str";
		  String str2 = "ing";
		 
		  String str3 = "str" + "ing";//常量池中的对象
		  String str4 = str1 + str2; //在堆上创建的新的对象	  
		  String str5 = "string";//常量池中的对象
		  System.out.println(str3 == str4);//false
		  System.out.println(str3 == str5);//true
		  System.out.println(str4 == str5);//false
```

[![img](https://camo.githubusercontent.com/6008db677de95edd009ce8dc9337f5bdf7de8d91/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545372541432541362545342542382542322545362538422542432545362538452541352e706e67)](https://camo.githubusercontent.com/6008db677de95edd009ce8dc9337f5bdf7de8d91/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545372541432541362545342542382542322545362538422542432545362538452541352e706e67)

**尽量避免多个字符串拼接，因为这样会重新创建对象。如果需要改变字符串的话，可以使用 StringBuilder 或者 StringBuffer。**

##### 3.2 String s1 = new String("abc")，这句话创建了几个字符串对象？

**将创建 1 或 2 个字符串。如果池中已存在字符串常量“abc”，则只会在堆空间创建一个字符串常量“abc”。如果池中没有字符串常量“abc”，那么它将首先在池中创建，然后在堆空间中创建，因此将创建总共 2 个字符串对象。**

**验证：**

```java
		String s1 = new String("abc");// 堆内存的地址值
		String s2 = "abc";
		System.out.println(s1 == s2);// 输出 false,因为一个是堆内存，一个是常量池的内存，故两者是不同的。
		System.out.println(s1.equals(s2));// 输出 true
		
结果：
false
true
```

##### 3.3 8 种基本类型的包装类和常量池

- **Java 基本类型的包装类的大部分都实现了常量池技术，即 Byte,Short,Integer,Long,Character,Boolean；这 5 种包装类默认创建了数值[-128，127] 的相应类型的缓存数据，但是超出此范围仍然会去创建新的对象。** 为啥把缓存设置为[-128，127]区间？（[参见issue/461](https://github.com/Snailclimb/JavaGuide/issues/461)）性能和资源之间的权衡。
- **两种浮点数类型的包装类 Float,Double 并没有实现常量池技术。**

```java
		Integer i1 = 33;
		Integer i2 = 33;
		System.out.println(i1 == i2);// 输出 true
		Integer i11 = 333;
		Integer i22 = 333;
		System.out.println(i11 == i22);// 输出 false
		Double i3 = 1.2;
		Double i4 = 1.2;
		System.out.println(i3 == i4);// 输出 false
```

**Integer 缓存源代码：**

```java
/**
*此方法将始终缓存-128 到 127（包括端点）范围内的值，并可以缓存此范围之外的其他值。
*/
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

**应用场景：**

1. Integer i1=40；Java 在编译的时候会直接将代码封装成 Integer i1=Integer.valueOf(40);，从而使用常量池中的对象。
2. Integer i1 = new Integer(40);这种情况下会创建新的对象。

```java
  Integer i1 = 40;
  Integer i2 = new Integer(40);
  System.out.println(i1==i2);//输出 false
```

**Integer 比较更丰富的一个例子:**

```java
  Integer i1 = 40;
  Integer i2 = 40;
  Integer i3 = 0;
  Integer i4 = new Integer(40);
  Integer i5 = new Integer(40);
  Integer i6 = new Integer(0);
  
  System.out.println("i1=i2   " + (i1 == i2));
  System.out.println("i1=i2+i3   " + (i1 == i2 + i3));
  System.out.println("i1=i4   " + (i1 == i4));
  System.out.println("i4=i5   " + (i4 == i5));
  System.out.println("i4=i5+i6   " + (i4 == i5 + i6));   
  System.out.println("40=i5+i6   " + (40 == i5 + i6));     
```

结果：

```java
i1=i2   true
i1=i2+i3   true
i1=i4   false
i4=i5   false
i4=i5+i6   true
40=i5+i6   true
```

解释：

语句 i4 == i5 + i6，因为+这个操作符不适用于 Integer 对象，首先 i5 和 i6 进行自动拆箱操作，进行数值相加，即 i4 == 40。然后 Integer 对象无法与数值进行直接比较，所以 i4 自动拆箱转为 int 值 40，最终这条语句转为 40 == 40 进行数值比较。





### 二、JVM 垃圾回收

#### 1. JVM 内存分配与回收？

Java 的自动内存管理主要是针对对象内存的回收和对象内存的分配。同时，Java 自动内存管理最核心的功能是 **堆** 内存中对象的分配与回收。

Java 堆是垃圾收集器管理的主要区域，因此也被称作**GC 堆（Garbage Collected Heap）**.从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：新生代和老年代：再细致一点有：Eden 空间、From Survivor、To Survivor 空间等。**进一步划分的目的是更好地回收内存，或者更快地分配内存。**

**堆空间的基本结构：**

[![img](https://camo.githubusercontent.com/4012482f49926b35d8557b63952ee605fd259f62/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33e5a086e7bb93e69e842e706e67)](https://camo.githubusercontent.com/4012482f49926b35d8557b63952ee605fd259f62/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d33e5a086e7bb93e69e842e706e67)

上图所示的 eden 区、s0("From") 区、s1("To") 区都属于新生代，tentired 区属于老年代。大部分情况，对象都会首先在 Eden 区域分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 s1("To")，并且对象的年龄还会加 1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。经过这次GC后，Eden区和"From"区已经被清空。这个时候，"From"和"To"会交换他们的角色，也就是新的"To"就是上次GC前的“From”，新的"From"就是上次GC前的"To"。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，"To"区被填满之后，会将所有对象移动到年老代中。

**堆内存常见的分配策略：**

1. 对象优先在Eden区域分配
2. 大对象直接进入老年代
3. 长期存活的对象将进入老年代

##### 1.1对象优先在Eden区域分配

目前主流的垃圾收集器都会采用分代回收算法，因此需要将堆内存分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

大多数情况下，对象在新生代中 eden 区分配。当 eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC.下面我们来进行实际测试以下。

在测试之前我们先来看看 **Minor GC 和 Full GC 有什么不同呢？**

- **新生代 GC（Minor GC）**:指发生新生代的的垃圾收集动作，Minor GC 非常频繁，回收速度一般也比较快。
- **老年代 GC（Major GC/Full GC）**:指发生在老年代的 GC，出现了 Major GC 经常会伴随至少一次的 Minor GC（并非绝对），Major GC 的速度一般会比 Minor GC 的慢 10 倍以上。

**测试：**

```Java
public class GCTest {

	public static void main(String[] args) {
		byte[] allocation1, allocation2;
		allocation1 = new byte[30900*1024];
		//allocation2 = new byte[900*1024];
	}
}
```

通过以下方式运行： [![img](https://camo.githubusercontent.com/2713b77464413e9aeb63cdd9d8cd198cb3fea34a/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32353137383335302e6a7067)](https://camo.githubusercontent.com/2713b77464413e9aeb63cdd9d8cd198cb3fea34a/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32353137383335302e6a7067)

添加的参数：`-XX:+PrintGCDetails` [![img](https://camo.githubusercontent.com/9c349ba9ea4778b4c5f7cd0fb7e7efd812902330/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f31303331373134362e6a7067)](https://camo.githubusercontent.com/9c349ba9ea4778b4c5f7cd0fb7e7efd812902330/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f31303331373134362e6a7067)

运行结果 (红色字体描述有误，应该是对应于 JDK1.7 的永久代)：

[![img](https://camo.githubusercontent.com/42464b598a233d4319009cc06e3d199b6331c314/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32383935343238362e6a7067)](https://camo.githubusercontent.com/42464b598a233d4319009cc06e3d199b6331c314/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32383935343238362e6a7067)

从上图我们可以看出 eden 区内存几乎已经被分配完全（即使程序什么也不做，新生代也会使用 2000 多 k 内存）。假如我们再为 allocation2 分配内存会出现什么情况呢？

```
allocation2 = new byte[900*1024];
```

[![img](https://camo.githubusercontent.com/215514817e1b01cd89ba942c382d2f64cb6c9e3d/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32383132383738352e6a7067)](https://camo.githubusercontent.com/215514817e1b01cd89ba942c382d2f64cb6c9e3d/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32362f32383132383738352e6a7067)

**简单解释一下为什么会出现这种情况：** 因为给 allocation2 分配内存的时候 eden 区内存几乎已经被分配完了，我们刚刚讲了当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC.GC 期间虚拟机又发现 allocation1 无法存入 Survivor 空间，所以只好通过 **分配担保机制** 把新生代的对象提前转移到老年代中去，老年代上的空间足够存放 allocation1，所以不会出现 Full GC。执行 Minor GC 后，后面分配的对象如果能够存在 eden 区的话，还是会在 eden 区分配内存。

##### 1.2 大对象直接进入老年代

大对象就是需要大量连续内存空间的对象（比如：字符串、数组）。

**为什么要这样呢？**

为了避免为大对象分配内存时由于分配担保机制带来的复制而降低效率。

##### 1.3 长期存活的对象将进入老年代

既然虚拟机采用了分代收集的思想来管理内存，那么内存回收时就必须能识别哪些对象应放在新生代，哪些对象应放在老年代中。为了做到这一点，虚拟机给每个对象一个对象年龄（Age）计数器。

如果对象在 Eden 出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间中，并将对象年龄设为 1.对象在 Survivor 中每熬过一次 MinorGC,年龄就增加 1 岁，当它的年龄增加到一定程度（默认为 15 岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。

##### 1.4 动态对象年龄判定

为了更好的适应不同程序的内存情况，虚拟机不是永远要求对象年龄必须达到了某个值才能进入老年代，如果 Survivor 空间中相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无需达到要求的年龄。



#### 2. JVM如何判断对象已经死亡？

堆中几乎放着所有的对象实例，对堆垃圾回收前的第一步就是要判断那些对象已经死亡（即不能再被任何途径使用的对象）。

[![img](https://camo.githubusercontent.com/b3a4bf00f50b9981e3c7b933d54f276e20933b66/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f31313033343235392e6a7067)](https://camo.githubusercontent.com/b3a4bf00f50b9981e3c7b933d54f276e20933b66/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f31313033343235392e6a7067)

##### 2.1 引用计数法

给对象中添加一个引用计数器，每当有一个地方引用它，计数器就加 1；当引用失效，计数器就减 1；任何时候计数器为 0 的对象就是不可能再被使用的。

**这个方法实现简单，效率高，但是目前主流的虚拟机中并没有选择这个算法来管理内存，其最主要的原因是它很难解决对象之间相互循环引用的问题。** 所谓对象之间的相互引用问题，如下面代码所示：除了对象 objA 和 objB 相互引用着对方之外，这两个对象之间再无任何引用。但是他们因为互相引用对方，导致它们的引用计数器都不为 0，于是引用计数算法无法通知 GC 回收器回收他们。

```java
public class ReferenceCountingGc {
    Object instance = null;
	public static void main(String[] args) {
		ReferenceCountingGc objA = new ReferenceCountingGc();
		ReferenceCountingGc objB = new ReferenceCountingGc();
		objA.instance = objB;
		objB.instance = objA;
		objA = null;
		objB = null;
	}
}
```

##### 2.2 可达性分析算法

这个算法的基本思想就是通过一系列的称为 **“GC Roots”** 的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的。

[![可达性分析算法 ](https://camo.githubusercontent.com/6c6a9c7e2a7849cab8d5966ec1916115380e2842/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f37323736323034392e6a7067)](https://camo.githubusercontent.com/6c6a9c7e2a7849cab8d5966ec1916115380e2842/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f37323736323034392e6a7067)

**在Java中，可作为GC Root的对象包括以下几种：**

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI（即一般说的Native方法）引用的对象

##### 2.3 不可达的对象并非“非死不可”

即使在可达性分析算法中不可达的对象，也并非是“非死不可”的，这时候它们暂时处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历再次标记过程。
**标记的前提是对象在进行可达性分析后发现没有与GC Roots相连接的引用链。**

1. 第一次标记并进行一次筛选

   筛选的条件是此对象是否有必要执行finalize()方法。
   当对象没有覆盖finalize方法，或者finzlize方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”，对象被回收。

2. 第二次标记

   如果这个对象被判定为有必要执行finalize（）方法，那么这个对象将会被放置在一个名为：F-Queue的队列之中，并在稍后由一条虚拟机自动建立的、低优先级的Finalizer线程去执行。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束。这样做的原因是，如果一个对象finalize（）方法中执行缓慢，或者发生死循环（更极端的情况），将很可能会导致F-Queue队列中的其他对象永久处于等待状态，甚至导致整个内存回收系统崩溃。
   Finalize（）方法是对象脱逃死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模标记，如果对象要在finalize（）中成功拯救自己----只要重新与引用链上的任何的一个对象建立关联即可，譬如把自己赋值给某个类变量或对象的成员变量，那在第二次标记时它将移除出“即将回收”的集合。如果对象这时候还没逃脱，那基本上它就真的被回收了。

##### 2.4 Java引用

无论是通过引用计数法判断对象引用数量，还是通过可达性分析法判断对象的引用链是否可达，判定对象的存活都与“引用”有关。

JDK1.2 之前，Java 中引用的定义很传统：如果 reference 类型的数据存储的数值代表的是另一块内存的起始地址，就称这块内存代表一个引用。

JDK1.2 以后，Java 对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用、虚引用四种（引用强度逐渐减弱）

1. **强引用（StrongReference）**

   强引用就是指在程序代码中普遍存在的，类似`Object obj = new Object()`这类似的引用，只要强引用在，垃圾搜集器永远不会搜集被引用的对象。也就是说，宁愿出现内存溢出，也不会回收这些对象。

2. **软引用（SoftReference）**

   软引用是用来描述一些有用但并不是必需的对象，在Java中用`java.lang.ref.SoftReference`类来表示。对于软引用关联着的对象，只有在内存不足的时候JVM才会回收该对象。因此，这一点可以很好地用来解决OOM的问题，并且这个特性很适合用来实现缓存：比如网页缓存、图片缓存等。

   ```java
   import java.lang.ref.SoftReference;
    
   public class Main {
       public static void main(String[] args) {
            
           SoftReference<String> sr = new SoftReference<String>(new String("hello"));
           System.out.println(sr.get());
       }
   }
   ```

   软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。

3. **弱引用（WeakReference）**

   弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在java中，用java.lang.ref.WeakReference类来表示。下面是使用示例：

   ```Java
   import java.lang.ref.WeakReference;
    
   public class Main {
       public static void main(String[] args) {
        
           WeakReference<String> sr = new WeakReference<String>(new String("hello"));
            
           System.out.println(sr.get());
           System.gc();                //通知JVM的gc进行垃圾回收
           System.out.println(sr.get());
       }
   }
   ```

   ***弱引用与软引用的区别在于***：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。

4. 如果一个对象只具有弱引用，那就类似于**可有可无的生活用品**。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。

   弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

5. **虚引用（PhantomReference）**

   虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用`java.lang.ref.PhantomReference`类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。
    要注意的是，虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

   ```Java
   import java.lang.ref.PhantomReference;
   import java.lang.ref.ReferenceQueue;
    
   public class Main {
       public static void main(String[] args) {
           ReferenceQueue<String> queue = new ReferenceQueue<String>();
           PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);
           System.out.println(pr.get());
       }
   }
   ```

   **虚引用与软引用和弱引用的一个区别在于：** 虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

   特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**。

   在使用软引用和弱引用的时候，我们可以显示地通过System.gc()来通知JVM进行垃圾回收，但是要注意的是，虽然发出了通知，JVM不一定会立刻执行，也就是说这句是无法确保此时JVM一定会进行垃圾回收的。

##### 2.5 如何判断一个常量是废弃常量

运行时常量池主要回收的是废弃的常量。那么，我们如何判断一个常量是废弃常量呢？

假如在常量池中存在字符串 "abc"，如果当前没有任何 String 对象引用该字符串常量的话，就说明常量 "abc" 就是废弃常量，如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池。

注意：我们在 [可能是把 Java 内存区域讲的最清楚的一篇文章 ](https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247484303&idx=1&sn=af0fd436cef755463f59ee4dd0720cbd&chksm=fd9855eecaefdcf8d94ac581cfda4e16c8a730bda60c3b50bc55c124b92f23b6217f7f8e58d5&token=506869459&lang=zh_CN#rd)也讲了 JDK1.7 及之后版本的 JVM 已经将运行时常量池从方法区中移了出来，在 Java 堆（Heap）中开辟了一块区域存放运行时常量池。

##### 2.6 如何判断一个类是无用的类

方法区主要回收的是无用的类，那么如何判断一个类是无用的类的呢？

判定一个常量是否是“废弃常量”比较简单，而要判定一个类是否是“无用的类”的条件则相对苛刻许多。类需要同时满足下面 3 个条件才能算是 **“无用的类”** ：

- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已经被回收。
- 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

虚拟机可以对满足上述 3 个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样不使用了就会必然被回收。

#### 3. 垃圾收集算法

[![垃圾收集算法分类](https://camo.githubusercontent.com/abe4956dea64d2661d54586797be1516f9166d0b/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352539452538332545352539432542452545362539342542362545392539422538362545372541452539372545362542332539352e6a7067)](https://camo.githubusercontent.com/abe4956dea64d2661d54586797be1516f9166d0b/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352539452538332545352539432542452545362539342542362545392539422538362545372541452539372545362542332539352e6a7067)

##### 3.1 标记-清除算法

该算法分为“标记”和“清除”阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。它是最基础的收集算法，后续的算法都是对其不足进行改进得到。这种垃圾收集算法会带来两个明显的问题：

1. **效率问题**
2. **空间问题（标记清除后会产生大量不连续的碎片）**

[![公众号](https://camo.githubusercontent.com/dc1f798e7c7f9aa9a3ab692db10a6b1788e5d505/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f36333730373238312e6a7067)](https://camo.githubusercontent.com/dc1f798e7c7f9aa9a3ab692db10a6b1788e5d505/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f36333730373238312e6a7067)

##### 3.2 复制算法

为了解决效率问题，“复制”收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

[![公众号](https://camo.githubusercontent.com/94cfc5e1fbe9d49b3ed056d2943fd86dac1833a2/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f39303938343632342e6a7067)](https://camo.githubusercontent.com/94cfc5e1fbe9d49b3ed056d2943fd86dac1833a2/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f39303938343632342e6a7067)

##### 3.3 标记-整理算法

根据老年代的特点提出的一种标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

[![标记-整理算法 ](https://camo.githubusercontent.com/e5223ec7b2460498e1934c14eeaf969bafdcab59/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f39343035373034392e6a7067)](https://camo.githubusercontent.com/e5223ec7b2460498e1934c14eeaf969bafdcab59/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f39343035373034392e6a7067)

##### 3.4 分代收集算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

**比如在新生代中，每次收集都会有大量对象死去，所以可以选择复制算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**

**延伸面试问题：** HotSpot 为什么要分为新生代和老年代？



#### 4. 垃圾收集器

[![垃圾收集器分类](https://camo.githubusercontent.com/f769c52ae89d045695e8b41d4583dda4b921ca3e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352539452538332545352539432542452545362539342542362545392539422538362545352539392541382e6a7067)](https://camo.githubusercontent.com/f769c52ae89d045695e8b41d4583dda4b921ca3e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352539452538332545352539432542452545362539342542362545392539422538362545352539392541382e6a7067)

**如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。**

虽然我们对各个收集器进行比较，但并非要挑选出一个最好的收集器。因为直到现在为止还没有最好的垃圾收集器出现，更加没有万能的垃圾收集器，**我们能做的就是根据具体应用场景选择适合自己的垃圾收集器**。试想一下：如果有一种四海之内、任何场景下都适用的完美收集器存在，那么我们的 HotSpot 虚拟机就不会实现那么多不同的垃圾收集器了。

##### 4.1 Serial 收集器

Serial（串行）收集器收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的 **“单线程”** 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ **"Stop The World"** ），直到它收集结束。

**新生代采用复制算法，老年代采用标记-整理算法。** [![ Serial 收集器 ](https://camo.githubusercontent.com/aba41c5c08ea9884554b9a69ea69c7ceeebc83ff/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f34363837333032362e6a7067)](https://camo.githubusercontent.com/aba41c5c08ea9884554b9a69ea69c7ceeebc83ff/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f34363837333032362e6a7067)

虚拟机的设计者们当然知道 Stop The World 带来的不良用户体验，所以在后续的垃圾收集器设计中停顿时间在不断缩短（仍然还有停顿，寻找最优秀的垃圾收集器的过程仍然在继续）。

但是 Serial 收集器有没有优于其他垃圾收集器的地方呢？当然有，它**简单而高效（与其他收集器的单线程相比）**。Serial 收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。Serial 收集器对于运行在 Client 模式下的虚拟机来说是个不错的选择。

##### 4.2 ParNew 收集器

**ParNew 收集器其实就是 Serial 收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和 Serial 收集器完全一样。**

**新生代采用复制算法，老年代采用标记-整理算法。** [![ParNew 收集器 ](https://camo.githubusercontent.com/f298ba56ec4667487fdf4acc987f2ef9e6df254e/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f32323031383336382e6a7067)](https://camo.githubusercontent.com/f298ba56ec4667487fdf4acc987f2ef9e6df254e/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f32323031383336382e6a7067)

它是许多运行在 Server 模式下的虚拟机的首要选择，除了 Serial 收集器外，只有它能与 CMS 收集器（真正意义上的并发收集器，后面会介绍到）配合工作。

**并行和并发概念补充：**

- **并行（Parallel）** ：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
- **并发（Concurrent）**：指用户线程与垃圾收集线程同时执行（但不一定是并行，可能会交替执行），用户程序在继续运行，而垃圾收集器运行在另一个 CPU 上。

##### 4.3 Parallel Scavenge 收集器

Parallel Scavenge 收集器也是使用复制算法的多线程收集器，它看上去几乎和ParNew都一样。 **那么它有什么特别之处呢？**

```
-XX:+UseParallelGC 

    使用 Parallel 收集器+ 老年代串行

-XX:+UseParallelOldGC

    使用 Parallel 收集器+ 老年代并行
```

**Parallel Scavenge 收集器关注点是吞吐量（高效率的利用 CPU）。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值。** Parallel Scavenge 收集器提供了很多参数供用户找到最合适的停顿时间或最大吞吐量，如果对于收集器运作不太了解的话，手工优化存在困难的话可以选择把内存管理优化交给虚拟机去完成也是一个不错的选择。

**新生代采用复制算法，老年代采用标记-整理算法。** [![Parallel Scavenge 收集器 ](https://camo.githubusercontent.com/f298ba56ec4667487fdf4acc987f2ef9e6df254e/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f32323031383336382e6a7067)](https://camo.githubusercontent.com/f298ba56ec4667487fdf4acc987f2ef9e6df254e/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f32323031383336382e6a7067)

##### 4.4.Serial Old 收集器

**Serial 收集器的老年代版本**，它同样是一个单线程收集器。它主要有两大用途：一种用途是在 JDK1.5 以及以前的版本中与 Parallel Scavenge 收集器搭配使用，另一种用途是作为 CMS 收集器的后备方案。

##### 4.5 Parallel Old 收集器

**Parallel Scavenge 收集器的老年代版本**。使用多线程和“标记-整理”算法。在注重吞吐量以及 CPU 资源的场合，都可以优先考虑 Parallel Scavenge 收集器和 Parallel Old 收集器。

##### 4.6 CMS 收集器

**CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用。**

**CMS（Concurrent Mark Sweep）收集器是 HotSpot 虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。**

从名字中的**Mark Sweep**这两个词可以看出，CMS 收集器是一种 **“标记-清除”算法**实现的，它的运作过程相比于前面几种垃圾收集器来说更加复杂一些。整个过程分为四个步骤：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短
- **并发清除：** 开启用户线程，同时 GC 线程开始对为标记的区域做清扫。

[![CMS 垃圾收集器 ](https://camo.githubusercontent.com/d74545af0f987d17b7f41a60bb237a113fff925b/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f38323832353037392e6a7067)](https://camo.githubusercontent.com/d74545af0f987d17b7f41a60bb237a113fff925b/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d382d32372f38323832353037392e6a7067)

从它的名字就可以看出它是一款优秀的垃圾收集器，主要优点：**并发收集、低停顿**。但是它有下面三个明显的缺点：

- **对 CPU 资源敏感；**
- **无法处理浮动垃圾；**
- **它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。**

##### 4.7 G1 收集器

**G1 (Garbage-First) 是一款面向服务器的垃圾收集器,主要针对配备多颗处理器及大容量内存的机器. 以极高概率满足 GC 停顿时间要求的同时,还具备高吞吐量性能特征.**

被视为 JDK1.7 中 HotSpot 虚拟机的一个重要进化特征。它具备一下特点：

- **并行与并发**：G1 能充分利用 CPU、多核环境下的硬件优势，使用多个 CPU（CPU 或者 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程执行的 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
- **分代收集**：虽然 G1 可以不需要其他收集器配合就能独立管理整个 GC 堆，但是还是保留了分代的概念。
- **空间整合**：与 CMS 的“标记--清理”算法不同，G1 从整体来看是基于“标记整理”算法实现的收集器；从局部上来看是基于“复制”算法实现的。
- **可预测的停顿**：这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为 M 毫秒的时间片段内。

G1 收集器的运作大致分为以下几个步骤：

- **初始标记**
- **并发标记**
- **最终标记**
- **筛选回收**

**G1 收集器在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来)**。这种使用 Region 划分内存空间以及有优先级的区域回收方式，保证了 GF 收集器在有限时间内可以尽可能高的收集效率（把内存化整为零）。





### 三、JDK 监控和故障处理工具总结

#### 1.JDK 命令行工具

- **jps** (JVM Process Status）: 类似 UNIX 的 `ps` 命令。用户查看所有 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息
- **jstat**（ JVM Statistics Monitoring Tool）: 用于收集 HotSpot 虚拟机各方面的运行数据
- **jinfo** (Configuration Info for Java) : Configuration Info forJava,显示虚拟机配置信息
- **jmap** (Memory Map for Java) :生成堆转储快照
- **jhat** (JVM Heap Dump Browser ) : 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果
- **jstack** (Stack Trace for Java):生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合

###### 1.1 `jps`:查看所有 Java 进程

`jps`：显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一 ID（Local Virtual Machine Identifier,LVMID）。`jps -q`：只输出进程的本地虚拟机唯一 ID。

```Java
C:\Users\SnailClimb>jps
7360 NettyClient2
17396
7972 Launcher
16504 Jps
17340 NettyServer
```

`jps -l`:输出主类的全名，如果进程执行的是 Jar 包，输出 Jar 路径。

```Java
C:\Users\SnailClimb>jps -l
7360 firstNettyDemo.NettyClient2
17396
7972 org.jetbrains.jps.cmdline.Launcher
16492 sun.tools.jps.Jps
17340 firstNettyDemo.NettyServer
```

`jps -v`：输出虚拟机进程启动时 JVM 参数。

`jps -m`：输出传递给 Java 进程 main() 函数的参数。



###### 1.2 `jstat`: 监视虚拟机各种运行状态信息

jstat（JVM Statistics Monitoring Tool） 使用于监视虚拟机各种运行状态信息的命令行工具。 它可以显示本地或者远程（需要远程主机提供 RMI 支持）虚拟机进程中的类信息、内存、垃圾收集、JIT 编译等运行数据，在没有 GUI，只提供了纯文本控制台环境的服务器上，它将是运行期间定位虚拟机性能问题的首选工具。

**jstat 命令使用格式：**

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

比如 `jstat -gc -h3 31736 1000 10`表示分析进程 id 为 31736 的 gc 情况，每隔 1000ms 打印一次记录，打印 10 次停止，每 3 行后打印指标头部。

**常见的 option 如下：**

- `jstat -class vmid` ：显示 ClassLoader 的相关信息；
- `jstat -compiler vmid` ：显示 JIT 编译的相关信息；
- `jstat -gc vmid` ：显示与 GC 相关的堆信息；
- `jstat -gccapacity vmid` ：显示各个代的容量及使用情况；
- `jstat -gcnew vmid` ：显示新生代信息；
- `jstat -gcnewcapcacity vmid` ：显示新生代大小与使用情况；
- `jstat -gcold vmid` ：显示老年代和永久代的信息；
- `jstat -gcoldcapacity vmid` ：显示老年代的大小；
- `jstat -gcpermcapacity vmid` ：显示永久代大小；
- `jstat -gcutil vmid` ：显示垃圾收集信息；

另外，加上 `-t`参数可以在输出信息上加一个 Timestamp 列，显示程序的运行时间。



###### 1.3 `jinfo`: 实时地查看和调整虚拟机各项参数

`jinfo vmid` :输出当前 jvm 进程的全部参数和系统属性 (第一部分是系统的属性，第二部分是 JVM 的参数)。

`jinfo -flag name vmid` :输出对应名称的参数的具体值。比如输出 MaxHeapSize、查看当前 jvm 进程是否开启打印 GC 日志 ( `-XX:PrintGCDetails` :详细 GC 日志模式，这两个都是默认关闭的)。

```Java
C:\Users\SnailClimb>jinfo  -flag MaxHeapSize 17340
-XX:MaxHeapSize=2124414976
C:\Users\SnailClimb>jinfo  -flag PrintGC 17340
-XX:-PrintGC
```

使用 jinfo 可以在不重启虚拟机的情况下，可以动态的修改 jvm 的参数。尤其在线上的环境特别有用,请看下面的例子：

`jinfo -flag [+|-]name vmid` 开启或者关闭对应名称的参数。

```Java
C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:-PrintGC

C:\Users\SnailClimb>jinfo  -flag  +PrintGC 17340

C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:+PrintGC
```

### 

###### 1.4 `jmap`:生成堆转储快照

`jmap`（Memory Map for Java）命令用于生成堆转储快照。 如果不使用 `jmap` 命令，要想获取 Java 堆转储，可以使用 `“-XX:+HeapDumpOnOutOfMemoryError”` 参数，可以让虚拟机在 OOM 异常出现之后自动生成 dump 文件，Linux 命令下可以通过 `kill -3` 发送进程退出信号也能拿到 dump 文件。

`jmap` 的作用并不仅仅是为了获取 dump 文件，它还可以查询 finalizer 执行队列、Java 堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。和`jinfo`一样，`jmap`有不少功能在 Windows 平台下也是受限制的。

示例：将指定应用程序的堆快照输出到桌面。后面，可以通过 jhat、Visual VM 等工具分析该堆文件。

```java
C:\Users\SnailClimb>jmap -dump:format=b,file=C:\Users\SnailClimb\Desktop\heap.hprof 17340
Dumping heap to C:\Users\SnailClimb\Desktop\heap.hprof ...
Heap dump file created
```



###### **1.5 jhat**: 分析 heapdump 文件

**jhat** 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果。

```
C:\Users\SnailClimb>jhat C:\Users\SnailClimb\Desktop\heap.hprof
Reading from C:\Users\SnailClimb\Desktop\heap.hprof...
Dump file created Sat May 04 12:30:31 CST 2019
Snapshot read, resolving...
Resolving 131419 objects...
Chasing references, expect 26 dots..........................
Eliminating duplicate references..........................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```



###### **1.6 jstack** :生成虚拟机当前时刻的线程快照

`jstack`（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合.

生成线程快照的目的主要是定位线程长时间出现停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过`jstack`来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者在等待些什么资源。

**下面是一个线程死锁的代码。我们下面会通过 jstack 命令进行死锁检查，输出死锁信息，找到发生死锁的线程。**

```java
public class DeadLockDemo {
    private static Object resource1 = new Object();//资源 1
    private static Object resource2 = new Object();//资源 2

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (resource1) {
                System.out.println(Thread.currentThread() + "get resource1");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource2");
                synchronized (resource2) {
                    System.out.println(Thread.currentThread() + "get resource2");
                }
            }
        }, "线程 1").start();

        new Thread(() -> {
            synchronized (resource2) {
                System.out.println(Thread.currentThread() + "get resource2");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource1");
                synchronized (resource1) {
                    System.out.println(Thread.currentThread() + "get resource1");
                }
            }
        }, "线程 2").start();
    }
}
```

Output

```java
Thread[线程 1,5,main]get resource1
Thread[线程 2,5,main]get resource2
Thread[线程 1,5,main]waiting get resource2
Thread[线程 2,5,main]waiting get resource1
```

线程 A 通过 synchronized (resource1) 获得 resource1 的监视器锁，然后通过` Thread.sleep(1000);`让线程 A 休眠 1s 为的是让线程 B 得到执行然后获取到 resource2 的监视器锁。线程 A 和线程 B 休眠结束了都开始企图请求获取对方的资源，然后这两个线程就会陷入互相等待的状态，这也就产生了死锁。

**通过 jstack 命令分析：**

```java
C:\Users\SnailClimb>jps
13792 KotlinCompileDaemon
7360 NettyClient2
17396
7972 Launcher
8932 Launcher
9256 DeadLockDemo
10764 Jps
17340 NettyServer

C:\Users\SnailClimb>jstack 9256
```

输出的部分内容如下：

```java
Found one Java-level deadlock:
=============================
"线程 2":
  waiting to lock monitor 0x000000000333e668 (object 0x00000000d5efe1c0, a java.lang.Object),
  which is held by "线程 1"
"线程 1":
  waiting to lock monitor 0x000000000333be88 (object 0x00000000d5efe1d0, a java.lang.Object),
  which is held by "线程 2"

Java stack information for the threads listed above:
===================================================
"线程 2":
        at DeadLockDemo.lambda$main$1(DeadLockDemo.java:31)
        - waiting to lock <0x00000000d5efe1c0> (a java.lang.Object)
        - locked <0x00000000d5efe1d0> (a java.lang.Object)
        at DeadLockDemo$$Lambda$2/1078694789.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
"线程 1":
        at DeadLockDemo.lambda$main$0(DeadLockDemo.java:16)
        - waiting to lock <0x00000000d5efe1d0> (a java.lang.Object)
        - locked <0x00000000d5efe1c0> (a java.lang.Object)
        at DeadLockDemo$$Lambda$1/1324119927.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

可以看到 `jstack` 命令已经帮我们找到发生死锁的线程的具体信息。



#### 2.JDK 可视化分析工具

##### JConsole:Java 监视与管理控制台

JConsole 是基于 JMX 的可视化监视、管理工具。可以很方便的监视本地及远程服务器的 java 进程的内存使用情况。你可以在控制台输出`jconsole`命令启动或者在 JDK 目录下的 bin 目录找到`jconsole.exe`然后双击启动。

**连接 Jconsole**

[![连接 Jconsole](https://camo.githubusercontent.com/a108b0807e218d06448dd7b3b86821299b498fcf/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f314a436f6e736f6c652545382542462539452545362538452541352e706e67)](https://camo.githubusercontent.com/a108b0807e218d06448dd7b3b86821299b498fcf/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f314a436f6e736f6c652545382542462539452545362538452541352e706e67)

如果需要使用 JConsole 连接远程进程，可以在远程 Java 程序启动时加上下面这些参数:

```
-Djava.rmi.server.hostname=外网访问 ip 地址 
-Dcom.sun.management.jmxremote.port=60001   //监控的端口号
-Dcom.sun.management.jmxremote.authenticate=false   //关闭认证
-Dcom.sun.management.jmxremote.ssl=false
```

在使用 JConsole 连接时，远程进程地址如下：

```
外网访问 ip 地址:60001 
```

**查看 Java 程序概况**

[![查看 Java 程序概况 ](https://camo.githubusercontent.com/66f9893d2e4a18f63293f360393f11b2ad19269e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f322545362539462541352545372539432538424a6176612545372541382538422545352542412538462545362541362538322545352538362542352e706e67)](https://camo.githubusercontent.com/66f9893d2e4a18f63293f360393f11b2ad19269e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f322545362539462541352545372539432538424a6176612545372541382538422545352542412538462545362541362538322545352538362542352e706e67)

**内存监控**

JConsole 可以显示当前内存的详细信息。不仅包括堆内存/非堆内存的整体信息，还可以细化到 eden 区、survivor 区等的使用情况，如下图所示。

点击右边的“执行 GC(G)”按钮可以强制应用程序执行一个 Full GC。

> - **新生代 GC（Minor GC）**:指发生新生代的的垃圾收集动作，Minor GC 非常频繁，回收速度一般也比较快。
> - **老年代 GC（Major GC/Full GC）**:指发生在老年代的 GC，出现了 Major GC 经常会伴随至少一次的 Minor GC（并非绝对），Major GC 的速度一般会比 Minor GC 的慢 10 倍以上。

[![内存监控 ](https://camo.githubusercontent.com/67af8160bf285a050c33dc8688edfc382ba0ef4e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f332545352538362538352545352541442539382545372539422539312545362538452541372e706e67)](https://camo.githubusercontent.com/67af8160bf285a050c33dc8688edfc382ba0ef4e/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f332545352538362538352545352541442539382545372539422539312545362538452541372e706e67)

**线程监控**

类似我们前面讲的 `jstack` 命令，不过这个是可视化的。

最下面有一个"检测死锁 (D)"按钮，点击这个按钮可以自动为你找到发生死锁的线程以及它们的详细信息 。

[![线程监控 ](https://camo.githubusercontent.com/ce47fa82d3c750b97869c04513cd4f8b36f25ee5/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f342545372542412542462545372541382538422545372539422539312545362538452541372e706e67)](https://camo.githubusercontent.com/ce47fa82d3c750b97869c04513cd4f8b36f25ee5/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f342545372542412542462545372541382538422545372539422539312545362538452541372e706e67)



##### Visual VM:多合一故障处理工具

VisualVM 提供在 Java 虚拟机 (Java Virutal Machine, JVM) 上运行的 Java 应用程序的详细信息。在 VisualVM 的图形用户界面中，您可以方便、快捷地查看多个 Java 应用程序的相关信息。Visual VM 官网：https://visualvm.github.io/ 。Visual VM 中文文档:https://visualvm.github.io/documentation.html。

下面这段话摘自《深入理解 Java 虚拟机》。

> VisualVM（All-in-One Java Troubleshooting Tool）是到目前为止随 JDK 发布的功能最强大的运行监视和故障处理程序，官方在 VisualVM 的软件说明中写上了“All-in-One”的描述字样，预示着他除了运行监视、故障处理外，还提供了很多其他方面的功能，如性能分析（Profiling）。VisualVM 的性能分析功能甚至比起 JProfiler、YourKit 等专业且收费的 Profiling 工具都不会逊色多少，而且 VisualVM 还有一个很大的优点：不需要被监视的程序基于特殊 Agent 运行，因此他对应用程序的实际性能的影响很小，使得他可以直接应用在生产环境中。这个优点是 JProfiler、YourKit 等工具无法与之媲美的。

VisualVM 基于 NetBeans 平台开发，因此他一开始就具备了插件扩展功能的特性，通过插件扩展支持，VisualVM 可以做到：

- **显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。**
- **监视应用程序的 CPU、GC、堆、方法区以及线程的信息（jstat、jstack）。**
- **dump 以及分析堆转储快照（jmap、jhat）。**
- **方法级的程序运行性能分析，找到被调用最多、运行时间最长的方法。**
- **离线程序快照：收集程序的运行时配置、线程 dump、内存 dump 等信息建立一个快照，可以将快照发送开发者处进行 Bug 反馈。**
- **其他 plugins 的无限的可能性......**



### 四、类文件结构

在 Java 中，JVM 可以理解的代码就叫做`字节码`（即扩展名为 `.class` 的文件），它不面向任何特定的处理器，只面向虚拟机。Java 语言通过字节码的方式，在一定程度上解决了传统解释型语言执行效率低的问题，同时又保留了解释型语言可移植的特点。所以 Java 程序运行时比较高效，而且，由于字节码并不针对一种特定的机器，因此，Java 程序无须重新编译便可在多种不同操作系统的计算机上运行。

Clojure（Lisp 语言的一种方言）、Groovy、Scala 等语言都是运行在 Java 虚拟机之上。下图展示了不同的语言被不同的编译器编译成`.class`文件最终运行在 Java 虚拟机之上。`.class`文件的二进制格式可以使用 [WinHex](https://www.x-ways.net/winhex/) 查看。

[![Java虚拟机](https://camo.githubusercontent.com/13e15128fdac41fa66dfa608cd6ca3632f6bcc92/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f332f313633323562386531393066626162643f773d37313226683d33313626663d706e6726733d3137363433)](https://camo.githubusercontent.com/13e15128fdac41fa66dfa608cd6ca3632f6bcc92/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f352f332f313633323562386531393066626162643f773d37313226683d33313626663d706e6726733d3137363433)

**可以说.class文件是不同的语言在 Java 虚拟机之间的重要桥梁，同时也是支持 Java 跨平台很重要的一个原因。**

#### 1.Class 文件结构总结

根据 Java 虚拟机规范，类文件由单个 ClassFile 结构组成：

```java
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    field_info     fields[fields_count];//一个类会可以有个字段
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
}
```

下面详细介绍一下 Class 文件结构涉及到的一些组件。

**Class文件字节码结构组织示意图** （之前在网上保存的，非常不错，原出处不明）：

[![类文件字节码结构组织示意图](https://camo.githubusercontent.com/b500a94627f72a03fd9ce4063cea156512110ade/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545372542312542422545362539362538372545342542422542362545352541442539372545382538412538322545372541302538312545372542422539332545362539452538342545372542422538342545372542422538372545372541342542412545362538342538462545352539422542452e706e67)](https://camo.githubusercontent.com/b500a94627f72a03fd9ce4063cea156512110ade/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545372542312542422545362539362538372545342542422542362545352541442539372545382538412538322545372541302538312545372542422539332545362539452538342545372542422538342545372542422538372545372541342542412545362538342538462545352539422542452e706e67)

### 

##### 1.1 魔数

```Java
    u4             magic; //Class 文件的标志
```

每个 Class 文件的头四个字节称为魔数（Magic Number）,它的唯一作用是**确定这个文件是否为一个能被虚拟机接收的 Class 文件**。

程序设计者很多时候都喜欢用一些特殊的数字表示固定的文件类型或者其它特殊的含义。

##### 1.2 Class 文件版本

```Java
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
```

紧接着魔数的四个字节存储的是 Class 文件的版本号：第五和第六是**次版本号**，第七和第八是**主版本号**。

高版本的 Java 虚拟机可以执行低版本编译器生成的 Class 文件，但是低版本的 Java 虚拟机不能执行高版本编译器生成的 Class 文件。所以，我们在实际开发的时候要确保开发的的 JDK 版本和生产环境的 JDK 版本保持一致。

##### 1.3 常量池

```java
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
```

紧接着主次版本号之后的是常量池，常量池的数量是 constant_pool_count-1（**常量池计数器是从1开始计数的，将第0项常量空出来是有特殊考虑的，索引值为0代表“不引用任何一个常量池项”**）。

常量池主要存放两大常量：字面量和符号引用。字面量比较接近于 Java 语言层面的的常量概念，如文本字符串、声明为 final 的常量值等。而符号引用则属于编译原理方面的概念。包括下面三类常量：

- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符

常量池中每一项常量都是一个表，这14种表有一个共同的特点：**开始的第一位是一个 u1 类型的标志位 -tag 来标识常量的类型，代表当前这个常量属于哪种常量类型．**

| 类型                             | 标志（tag） | 描述                   |
| -------------------------------- | ----------- | ---------------------- |
| CONSTANT_utf8_info               | 1           | UTF-8编码的字符串      |
| CONSTANT_Integer_info            | 3           | 整形字面量             |
| CONSTANT_Float_info              | 4           | 浮点型字面量           |
| CONSTANT_Long_info               | ５          | 长整型字面量           |
| CONSTANT_Double_info             | ６          | 双精度浮点型字面量     |
| CONSTANT_Class_info              | ７          | 类或接口的符号引用     |
| CONSTANT_String_info             | ８          | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | ９          | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10          | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11          | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12          | 字段或方法的符号引用   |
| CONSTANT_MothodType_info         | 16          | 标志方法类型           |
| CONSTANT_MethodHandle_info       | 15          | 表示方法句柄           |
| CONSTANT_InvokeDynamic_info      | 18          | 表示一个动态方法调用点 |

`.class` 文件可以通过`javap -v class类名` 指令来看一下其常量池中的信息(`javap -v class类名-> temp.txt` ：将结果输出到 temp.txt 文件)。

##### 1.4 访问标志

在常量池结束之后，紧接着的两个字节代表访问标志，这个标志用于识别一些类或者接口层次的访问信息，包括：这个 Class 是类还是接口，是否为 public 或者 abstract 类型，如果是类的话是否声明为 final 等等。

类访问和属性修饰符:

[![类访问和属性修饰符](https://camo.githubusercontent.com/d0969f48703bbba1434844b3f88f1319264ba0bf/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545382541452542462545392539372541452545362541302538372545352542462539372e706e67)](https://camo.githubusercontent.com/d0969f48703bbba1434844b3f88f1319264ba0bf/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545382541452542462545392539372541452545362541302538372545352542462539372e706e67)

我们定义了一个 Employee 类

```Java
package top.snailclimb.bean;
public class Employee {
   ...
}
```

通过`javap -v class类名` 指令来看一下类的访问标志。

[![查看类的访问标志](https://camo.githubusercontent.com/bf262f108bdd85a5749576bd7b6c8a92d88a83df/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539462541352545372539432538422545372542312542422545372539412538342545382541452542462545392539372541452545362541302538372545352542462539372e706e67)](https://camo.githubusercontent.com/bf262f108bdd85a5749576bd7b6c8a92d88a83df/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539462541352545372539432538422545372542312542422545372539412538342545382541452542462545392539372541452545362541302538372545352542462539372e706e67)

##### 1.5 当前类索引,父类索引与接口索引集合

```Java
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个雷可以实现多个接口
```

**类索引用于确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名，由于 Java 语言的单继承，所以父类索引只有一个，除了 java.lang.Object 之外，所有的 java 类都有父类，因此除了 java.lang.Object 外，所有 Java 类的父类索引都不为 0。**

**接口索引集合用来描述这个类实现了那些接口，这些被实现的接口将按implents(如果这个类本身是接口的话则是extends) 后的接口顺序从左到右排列在接口索引集合中。**

##### 1.6 字段表集合

```Java
    u2             fields_count;//Class 文件的字段的个数
    field_info     fields[fields_count];//一个类会可以有个字段
```

字段表（field info）用于描述接口或类中声明的变量。字段包括类级变量以及实例变量，但不包括在方法内部声明的局部变量。

**field info(字段表) 的结构:**

[![字段表的结构 ](https://camo.githubusercontent.com/1cad0f3668b8270bcbbda04647a4eb1e43a7cbc6/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545362541452542352545382541312541382545372539412538342545372542422539332545362539452538342e706e67)](https://camo.githubusercontent.com/1cad0f3668b8270bcbbda04647a4eb1e43a7cbc6/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545362541452542352545382541312541382545372539412538342545372542422539332545362539452538342e706e67)

- **access_flags:** 字段的作用域（`public` ,`private`,`protected`修饰符），是实例变量还是类变量（`static`修饰符）,可否被序列化（transient 修饰符）,可变性（final）,可见性（volatile 修饰符，是否强制从主内存读写）。
- **name_index:** 对常量池的引用，表示的字段的名称；
- **descriptor_index:** 对常量池的引用，表示字段和方法的描述符；
- **attributes_count:** 一个字段还会拥有一些额外的属性，attributes_count 存放属性的个数；
- **attributes[attributes_count]:** 存放具体属性具体内容。

上述这些信息中，各个修饰符都是布尔值，要么有某个修饰符，要么没有，很适合使用标志位来表示。而字段叫什么名字、字段被定义为什么数据类型这些都是无法固定的，只能引用常量池中常量来描述。

**字段的 access_flags 的取值:**

[![字段的access_flags的取值](https://camo.githubusercontent.com/66247c341a1faffa14aec0a92d222aaa90cd6fd1/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545362541452542352545372539412538346163636573735f666c6167732545372539412538342545352538462539362545352538302542432e706e67)](https://camo.githubusercontent.com/66247c341a1faffa14aec0a92d222aaa90cd6fd1/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545352541442539372545362541452542352545372539412538346163636573735f666c6167732545372539412538342545352538462539362545352538302542432e706e67)

##### 1.7 方法表集合

```Java
    u2             methods_count;//Class 文件的方法的数量
    method_info    methods[methods_count];//一个类可以有个多个方法
```

methods_count 表示方法的数量，而 method_info 表示的方法表。

Class 文件存储格式中对方法的描述与对字段的描述几乎采用了完全一致的方式。方法表的结构如同字段表一样，依次包括了访问标志、名称索引、描述符索引、属性表集合几项。

**method_info(方法表的) 结构:**

[![方法表的结构](https://camo.githubusercontent.com/03e6436571ebbf65a41f2bf8eeda39fc3fe29646/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539362542392545362542332539352545382541312541382545372539412538342545372542422539332545362539452538342e706e67)](https://camo.githubusercontent.com/03e6436571ebbf65a41f2bf8eeda39fc3fe29646/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539362542392545362542332539352545382541312541382545372539412538342545372542422539332545362539452538342e706e67)

**方法表的 access_flag 取值：**

[![方法表的 access_flag 取值](https://camo.githubusercontent.com/1bcb16ea2bf8fc0787d0883a6b2de3b009075339/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539362542392545362542332539352545382541312541382545372539412538346163636573735f666c61672545372539412538342545362538392538302545362539432538392545362541302538372545352542462539372545342542442538442e706e67)](https://camo.githubusercontent.com/1bcb16ea2bf8fc0787d0883a6b2de3b009075339/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f2545362539362542392545362542332539352545382541312541382545372539412538346163636573735f666c61672545372539412538342545362538392538302545362539432538392545362541302538372545352542462539372545342542442538442e706e67)

注意：因为`volatile`修饰符和`transient`修饰符不可以修饰方法，所以方法表的访问标志中没有这两个对应的标志，但是增加了`synchronized`、`native`、`abstract`等关键字修饰方法，所以也就多了这些关键字对应的标志。

##### 1.8 属性表集合

```Java
   u2             attributes_count;//此类的属性表中的属性数
   attribute_info attributes[attributes_count];//属性表集合
```

在 Class 文件，字段表，方法表中都可以携带自己的属性表集合，以用于描述某些场景专有的信息。与 Class 文件中其它的数据项目要求的顺序、长度和内容不同，属性表集合的限制稍微宽松一些，不再要求各个属性表具有严格的顺序，并且只要不与已有的属性名重复，任何人实现的编译器都可以向属性表中写 入自己定义的属性信息，Java 虚拟机运行时会忽略掉它不认识的属性。





































# 数据结构

## 一、树

### 1.二叉树

#### 1.1 定义

**二叉树**是n(n>=0)个结点的有限集合，该集合或者为空集（称为空二叉树），或者由一个根结点和两棵互不相交的、分别称为根结点的左子树和右子树组成。
 图1.1展示了一棵普通二叉树：



![img](https:////upload-images.jianshu.io/upload_images/7043118-797eb7ba417745b2.png?imageMogr2/auto-orient/strip|imageView2/2/w/455/format/webp)

图1.1 二叉树



#### 1.2 二叉树特点

由二叉树定义以及图示分析得出二叉树有以下特点：
 1）每个结点最多有两颗子树，所以二叉树中不存在度大于2的结点。
 2）左子树和右子树是有顺序的，次序不能任意颠倒。
 3）即使树中某结点只有一棵子树，也要区分它是左子树还是右子树。

#### 1.3 二叉树性质

1）在二叉树的第i层上最多有2i-1 个节点 。（i>=1）
 2）二叉树中如果深度为k,那么最多有2k-1个节点。(k>=1）
 3）n0=n2+1  n0表示度数为0的节点数，n2表示度数为2的节点数。
 4）在完全二叉树中，具有n个节点的完全二叉树的深度为[log2n]+1，其中[log2n]是向下取整。
 5）若对含 n 个结点的完全二叉树从上到下且从左至右进行 1 至 n 的编号，则对完全二叉树中任意一个编号为 i 的结点有如下特性：

> (1) 若 i=1，则该结点是二叉树的根，无双亲, 否则，编号为 [i/2] 的结点为其双亲结点;
>  (2) 若 2i>n，则该结点无左孩子，  否则，编号为 2i 的结点为其左孩子结点；
>  (3) 若 2i+1>n，则该结点无右孩子结点，  否则，编号为2i+1 的结点为其右孩子结点。

#### 1.4 斜树

**斜树**：所有的结点都只有左子树的二叉树叫左斜树。所有结点都是只有右子树的二叉树叫右斜树。这两者统称为斜树。




![img](https:////upload-images.jianshu.io/upload_images/7043118-a512316455261ec7.png?imageMogr2/auto-orient/strip|imageView2/2/w/373/format/webp)

图1.2 左斜树





![img](https:////upload-images.jianshu.io/upload_images/7043118-352190ff8558efcb.png?imageMogr2/auto-orient/strip|imageView2/2/w/342/format/webp)

图1.3 右斜树



#### 1.5 满二叉树

**满二叉树**：在一棵二叉树中。如果所有分支结点都存在左子树和右子树，并且所有叶子都在同一层上，这样的二叉树称为满二叉树。
 满二叉树的特点有：
 1）叶子只能出现在最下一层。出现在其它层就不可能达成平衡。
 2）非叶子结点的度一定是2。
 3）在同样深度的二叉树中，满二叉树的结点个数最多，叶子数最多。



![img](https:////upload-images.jianshu.io/upload_images/7043118-c7a557dda4ffc7da.png?imageMogr2/auto-orient/strip|imageView2/2/w/392/format/webp)

图1.4 满二叉树

#### 1.6 完全二叉树

**完全二叉树**：对一颗具有n个结点的二叉树按层编号，如果编号为i(1<=i<=n)的结点与同样深度的满二叉树中编号为i的结点在二叉树中位置完全相同，则这棵二叉树称为完全二叉树。
 图3.5展示一棵完全二叉树



![img](https:////upload-images.jianshu.io/upload_images/7043118-132fd0379f34bcc1.png?imageMogr2/auto-orient/strip|imageView2/2/w/404/format/webp)

图1.5 完全二叉树


**特点**

1）叶子结点只能出现在最下层和次下层。
2）最下层的叶子结点集中在树的左部。
3）倒数第二层若存在叶子结点，一定在右部连续位置。
4）如果结点度为1，则该结点只有左孩子，即没有右子树。
5）同样结点数目的二叉树，完全二叉树深度最小。
**注**：满二叉树一定是完全二叉树，但反过来不一定成立。

#### 1.7 二叉树的存储结构

##### 1.7.1 顺序存储

二叉树的顺序存储结构就是使用一维数组存储二叉树中的结点，并且结点的存储位置，就是数组的下标索引。





![img](https:////upload-images.jianshu.io/upload_images/7043118-3293242769696303.png?imageMogr2/auto-orient/strip|imageView2/2/w/441/format/webp)

图3.6

图3.6所示的一棵完全二叉树采用顺序存储方式，如图3.7表示：





![img](https:////upload-images.jianshu.io/upload_images/7043118-e916580c061a1139.png?imageMogr2/auto-orient/strip|imageView2/2/w/596/format/webp)

图3.7 顺序存储



由图3.7可以看出，当二叉树为完全二叉树时，结点数刚好填满数组。
 那么当二叉树不为完全二叉树时，采用顺序存储形式如何呢？例如：对于图3.8描述的二叉树：





![img](https:////upload-images.jianshu.io/upload_images/7043118-92d8a8d61c2aace7.png?imageMogr2/auto-orient/strip|imageView2/2/w/440/format/webp)

图3.8.png


 其中浅色结点表示结点不存在。那么图3.8所示的二叉树的顺序存储结构如图3.9所示：



![img](https:////upload-images.jianshu.io/upload_images/7043118-d6cd02856b386d6d.png?imageMogr2/auto-orient/strip|imageView2/2/w/448/format/webp)

图3.9

其中，∧表示数组中此位置没有存储结点。此时可以发现，顺序存储结构中已经出现了空间浪费的情况。
 那么对于图3.3所示的右斜树极端情况对应的顺序存储结构如图3.10所示：





![img](https:////upload-images.jianshu.io/upload_images/7043118-0ada42b04e0861a8.png?imageMogr2/auto-orient/strip|imageView2/2/w/700/format/webp)

图3.10



由图3.10可以看出，对于这种右斜树极端情况，采用顺序存储的方式是十分浪费空间的。因此，顺序存储一般适用于完全二叉树。

##### 1.7.2 二叉链表

既然顺序存储不能满足二叉树的存储需求，那么考虑采用链式存储。由二叉树定义可知，二叉树的每个结点最多有两个孩子。因此，可以将结点数据结构定义为一个数据和两个指针域。表示方式如图3.11所示：





![img](https:////upload-images.jianshu.io/upload_images/7043118-95cd18e8cc20316e.png?imageMogr2/auto-orient/strip|imageView2/2/w/315/format/webp)

图3.11

定义结点代码：

```cpp
typedef struct BiTNode{
    TElemType data;//数据
    struct BiTNode *lchild, *rchild;//左右孩子指针
} BiTNode, *BiTree;
```

则图3.6所示的二叉树可以采用图3.12表示。





![img](https:////upload-images.jianshu.io/upload_images/7043118-73ae201506a7adc9.png?imageMogr2/auto-orient/strip|imageView2/2/w/688/format/webp)

图3.12



图3.12中采用一种链表结构存储二叉树，这种链表称为二叉链表。

#### 1.8 二叉树遍历

二叉树的遍历一个重点考查的知识点。

##### 1.8.1 定义

**二叉树的遍历**是指从二叉树的根结点出发，按照某种次序依次访问二叉树中的所有结点，使得每个结点被访问一次，且仅被访问一次。
 二叉树的访问次序可以分为四种：

> 前序遍历
>  中序遍历
>  后序遍历
>  层序遍历

##### 1.8.2 前序遍历

**前序遍历**通俗的说就是从二叉树的根结点出发，当第一次到达结点时就输出结点数据，按照先向左在向右的方向访问。



![img](https:////upload-images.jianshu.io/upload_images/7043118-df454c0a574836de.png?imageMogr2/auto-orient/strip|imageView2/2/w/441/format/webp)

3.13


 图3.13所示二叉树访问如下：

> 从根结点出发，则第一次到达结点A，故输出A;
>  继续向左访问，第一次访问结点B，故输出B；
>  按照同样规则，输出D，输出H；
>  当到达叶子结点H，返回到D，此时已经是第二次到达D，故不在输出D，进而向D右子树访问，D右子树不为空，则访问至I，第一次到达I，则输出I；
>  I为叶子结点，则返回到D，D左右子树已经访问完毕，则返回到B，进而到B右子树，第一次到达E，故输出E；
>  向E左子树，故输出J；
>  按照同样的访问规则，继续输出C、F、G；

则3.13所示二叉树的前序遍历输出为：
 **ABDHIEJCFG**

##### 1.8.3 中序遍历

**中序遍历**就是从二叉树的根结点出发，当第二次到达结点时就输出结点数据，按照先向左在向右的方向访问。

图3.13所示二叉树中序访问如下：

> 从根结点出发，则第一次到达结点A，不输出A，继续向左访问，第一次访问结点B，不输出B；继续到达D，H；
>  到达H，H左子树为空，则返回到H，此时第二次访问H，故输出H；
>  H右子树为空，则返回至D，此时第二次到达D，故输出D；
>  由D返回至B，第二次到达B，故输出B；
>  按照同样规则继续访问，输出J、E、A、F、C、G；

则3.13所示二叉树的中序遍历输出为：
 **HDIBJEAFCG**

##### 1.8.4 后序遍历

**后序遍历**就是从二叉树的根结点出发，当第三次到达结点时就输出结点数据，按照先向左在向右的方向访问。

图3.13所示二叉树后序访问如下：

> 从根结点出发，则第一次到达结点A，不输出A，继续向左访问，第一次访问结点B，不输出B；继续到达D，H；
>  到达H，H左子树为空，则返回到H，此时第二次访问H，不输出H；
>  H右子树为空，则返回至H，此时第三次到达H，故输出H；
>  由H返回至D，第二次到达D，不输出D；
>  继续访问至I，I左右子树均为空，故第三次访问I时，输出I；
>  返回至D，此时第三次到达D，故输出D；
>  按照同样规则继续访问，输出J、E、B、F、G、C，A；

则图3.13所示二叉树的后序遍历输出为：
 **HIDJEBFGCA**
 虽然二叉树的遍历过程看似繁琐，但是由于二叉树是一种递归定义的结构，故采用递归方式遍历二叉树的代码十分简单。
 递归实现代码如下：

```cpp
/*二叉树的前序遍历递归算法*/
void PreOrderTraverse(BiTree T)
{
    if(T==NULL)
    return;
    printf("%c", T->data);  /*显示结点数据，可以更改为其他对结点操作*/
    PreOrderTraverse(T->lchild);    /*再先序遍历左子树*/
    PreOrderTraverse(T->rchild);    /*最后先序遍历右子树*/
}


/*二叉树的中序遍历递归算法*/
void InOrderTraverse(BiTree T)
{
    if(T==NULL)
    return;
    InOrderTraverse(T->lchild); /*中序遍历左子树*/
    printf("%c", T->data);  /*显示结点数据，可以更改为其他对结点操作*/
    InOrderTraverse(T->rchild); /*最后中序遍历右子树*/
}


/*二叉树的后序遍历递归算法*/
void PostOrderTraverse(BiTree T)
{
    if(T==NULL)
    return;
    PostOrderTraverse(T->lchild);   /*先后序遍历左子树*/
    PostOrderTraverse(T->rchild);   /*再后续遍历右子树*/
    printf("%c", T->data);  /*显示结点数据，可以更改为其他对结点操作*/
}
```

##### 1.8.5 层次遍历

层次遍历就是按照树的层次自上而下的遍历二叉树。针对图3.13所示二叉树的层次遍历结果为：
 **ABCDEFGHIJ**
 层次遍历的详细方法可以参考[二叉树的按层遍历法](https://blog.csdn.net/lingchen2348/article/details/52774535)。

##### 1.8.6 遍历常考考点

对于二叉树的遍历有一类典型题型。
 1）已知前序遍历序列和中序遍历序列，确定一棵二叉树。
 例题：若一棵二叉树的前序遍历为ABCDEF，中序遍历为CBAEDF，请画出这棵二叉树。
 分析：前序遍历第一个输出结点为根结点，故A为根结点。早中序遍历中根结点处于左右子树结点中间，故结点A的左子树中结点有CB，右子树中结点有EDF。
 如图3.14所示：





![img](https:////upload-images.jianshu.io/upload_images/7043118-8c94f437f66b5d44.png?imageMogr2/auto-orient/strip|imageView2/2/w/438/format/webp)

​                                                                        图3.14



按照同样的分析方法，对A的左右子树进行划分，最后得出二叉树的形态如图3.15所示：





![img](https:////upload-images.jianshu.io/upload_images/7043118-63b9acd9dc69201b.png?imageMogr2/auto-orient/strip|imageView2/2/w/381/format/webp)

​                                                                                 图3.15.png

2）已知后序遍历序列和中序遍历序列，确定一棵二叉树。
 后序遍历中最后访问的为根结点，因此可以按照上述同样的方法，找到根结点后分成两棵子树，进而继续找到子树的根结点，一步步确定二叉树的形态。
 **注**：已知前序遍历序列和后序遍历序列，不可以唯一确定一棵二叉树。



### 2.红黑树

#### **1.二叉搜索树**

二叉搜索树又叫二叉查找树或者二叉排序树，它首先是一个二叉树，而且必须满足下面的条件：

1）若左子树不空，则左子树上所有结点的值均小于它的根节点的值；

2）若右子树不空，则右子树上所有结点的值均大于它的根结点的值

3）左、右子树也分别为二叉搜索树





![img](https:////upload-images.jianshu.io/upload_images/4314397-9588c432805a9570.png?imageMogr2/auto-orient/strip|imageView2/2/w/300/format/webp)

#### **2.平衡二叉树**

二叉搜索树解决了许多问题，比如可以快速的查找最大值和最小值，可以快速找到排名第几位的值，快速搜索和排序等等。但普通的二叉搜索树有可能出现极不平衡的情况（斜树），这样我们的时间复杂度就有可能退化成 O(N) 的情况。比如我们现在插入的数据是 [1,2,3,4,5,6,7] 转换为二叉树如下：



![img](https:////upload-images.jianshu.io/upload_images/4314397-2f1facc1bcfc501c.png?imageMogr2/auto-orient/strip|imageView2/2/w/709/format/webp)

​                                                                          （斜树）

由于普通的二叉搜索树会出现极不平衡的情况，那么我们就必须得想想办法了，这个时候平衡二叉树就能帮到我们了。什么是平衡二叉树？平衡二叉搜索树（Self-balancing binary search tree）又被称为AVL树（有别于AVL算法），且具有以下性质：它是一 棵空树或**它的左右两个子树的高度差的绝对值不超过1**，并且左右两个子树都是一棵平衡二叉树。

平衡二叉树有一个很重要的性质：左右两个子树的高度差的绝对值不超过1。那么解决方案就是如果二叉树的左右高度超过 1 ，我们就把当前树调整为一棵平衡二叉树。这就涉及到**左旋**、**右旋**、**先右旋再左旋**、**先左旋再右旋**。

**2.1 右旋：**




![img](https:////upload-images.jianshu.io/upload_images/4314397-318ca114772ad3fc.png?imageMogr2/auto-orient/strip|imageView2/2/w/986/format/webp)



```swift
   TreeNode<K, V> *R_Rotation(TreeNode<K, V> *pNode) {
        TreeNode<K, V> *left = pNode->left;
        TreeNode<K, V> *right = left->right;
        left->right = pNode;
        pNode->left = right;
        // 重新调整高度
        pNode->height = max(getHeight(pNode->left), getHeight(pNode->right)) + 1;
        left->height = max(getHeight(left->left), getHeight(left->right)) + 1;
        return left;
    }
```

**2.2 左旋：**




![img](https:////upload-images.jianshu.io/upload_images/4314397-f06bb1d69571eaf0.png?imageMogr2/auto-orient/strip|imageView2/2/w/944/format/webp)

左旋



```swift
  TreeNode<K, V> *L_Rotation(TreeNode<K, V> *pNode) {
        TreeNode<K, V> *right = pNode->right;
        TreeNode<K, V> *left = right->left;
        right->left = pNode;
        pNode->right = left;
        // 重新调整高度
        pNode->height = max(getHeight(pNode->left), getHeight(pNode->right)) + 1;
        right->height = max(getHeight(right->left), getHeight(right->right)) + 1;
        return right;
    }
```

**2.3 先右旋再左旋：**




![img](https:////upload-images.jianshu.io/upload_images/4314397-d86683fad715ce89.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

先右旋再左旋



```php
  TreeNode<K, V> *R_L_Rotation(TreeNode<K, V> *pNode) {
    pNode->right = R_Rotation(pNode->right);
    return L_Rotation(pNode);
  }
```

**2.4 先左旋再右旋：**




![img](https:////upload-images.jianshu.io/upload_images/4314397-fc689643c778438f.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

先左旋再右旋



```php
  TreeNode<K, V> *L_R_Rotation(TreeNode<K, V> *pNode) {
    pNode->left = L_Rotation(pNode->left);
    return R_Rotation(pNode);
  }
```

#### **3.红黑树**

红黑树用法就比较广了，比如 JDK 1.8 的 HashMap，TreeMap，C++ 中的 map 和 multimap 等等。红黑树学习起来还是有一点难度的，这时如果我们心中有 B 树就有助于理解它，如果没有 B 树也没有关系。

红黑树的特性:
 （1）每个节点或者是黑色，或者是红色。
 （2）根节点是黑色。
 （3）每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
 （4）如果一个节点是红色的，则它的子节点必须是黑色的。
 （5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。





![img](https:////upload-images.jianshu.io/upload_images/4314397-a0bff7addb21cfe8.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/268/format/webp)

红黑树

假设我们现在要插入一个新的节点，如过插入的这个新的节点为黑色，那么必然会违反性质(5)，所以我们把新插入的点定义为红色的。但是如果插入的新节点为红色，就可能会违反性质(4) ，因此我们需要对其进行调整，使得整棵树依然满足红黑树的性质，也就是双红修正。接下来我们只要分情况分析就可以了：

1. 如果没有出现双红现象，父亲是黑色的不需要修正；
2. 叔叔是红色的 ，将叔叔和父亲染黑，然后爷爷染红；
3. 叔叔是黑色的，父亲是爷爷的左节点，且当前节点是其父节点的右孩子，将“父节点”作为“新的当前节点”，以“新的当前节点”为支点进行左旋。然后将“父节点”设为“黑色”，将“祖父节点”设为“红色”，以“祖父节点”为支点进行右旋；
4. 叔叔是黑色的，父亲是爷爷的左节点，且当前节点是其父节点的左孩子，将“父节点”设为“黑色”，将“祖父节点”设为“红色”，以“祖父节点”为支点进行右旋；
5. 叔叔是黑色的，父亲是爷爷的右节点，且当前节点是其父节点的左孩子，将“父节点”作为“新的当前节点”，以“新的当前节点”为支点进行右旋。然后将“父节点”设为“黑色”，将“祖父节点”设为“红色”，以“祖父节点”为支点进行左旋；
6. 叔叔是黑色的，父亲是爷爷的右节点，且当前节点是其父节点的右孩子，将“父节点”设为“黑色”，将“祖父节点”设为“红色”，以“祖父节点”为支点进行左旋；

上面的双红修正现象看似比较复杂，但实际上只有三种情况，一种是没有双红现象，另一种是父亲和叔叔都是红色的，最后一种是叔叔是黑色的。我们来画个实例看下：





![img](https:////upload-images.jianshu.io/upload_images/4314397-fae62fda145e9e35.png?imageMogr2/auto-orient/strip|imageView2/2/w/957/format/webp)



```php
void solveDoubleRed(TreeNode *pNode) {
        while (pNode->parent && pNode->parent->color == red) {// 情况 1
            TreeNode *uncle = brother(parent(pNode));

            if (getColor(uncle) == red) {// 情况2
                // 设置双亲和叔叔为黑色
                setColor(parent(pNode), black);
                setColor(uncle, black);
                // 指针回溯至爷爷
                pNode = parent(parent(pNode));
            } else {
                // 父亲是爷爷的左儿子
                if (parent(parent(pNode))->left = parent(pNode)) { // 情况 3 和 4
                    // 自己是父亲的右儿子
                    if (parent(pNode)->right == pNode) {
                        pNode = parent(pNode);
                        L_Rotation(pNode);
                    }
                    // 把我自己这边的红色节点挪到隔壁树上，但仍然不能违反性质 4 和 5
                    setColor(parent(pNode), black);
                    setColor(parent(parent(pNode)), red);
                    R_Rotation(parent(parent(pNode)));
                } else { // 情况 5 和 6
                    // 自己是父亲的左儿子
                    if (parent(pNode)->left == pNode) {
                        pNode = parent(pNode);
                        R_Rotation(pNode);
                    }
                    // 把我自己这边的红色节点挪到隔壁树上，但仍然不能违反性质 4 和 5
                    setColor(parent(pNode), black);
                    setColor(parent(parent(pNode)), red);
                    L_Rotation(parent(parent(pNode)));
                }
            }
        }

        // 根结点为黑色
        root->color = black;
    }
```

哎～好复杂这怎么记得住。如果要记住肯定不太可能而且费劲，接下来我们来分析下为什么要这么操作，还有没有更好的调整方法。我们所有的调整都是为了不违反性质4和性质5，假设我在左边的这个支树上新增了一个红色的节点，违反了性质4 。想法就是我把左支树上的一个红色节点，挪动右支树上去，这样就解决了我有两个连续红色节点的问题。但挪给右支树的过程中不能违反性质4和性质5，所以必须得考虑叔叔节点的颜色。



![img](https:////upload-images.jianshu.io/upload_images/4314397-1c1da607c0af9450.png?imageMogr2/auto-orient/strip|imageView2/2/w/1089/format/webp)



最后我们来看下红黑树的删除操作，红黑树的删除操作要比新增操作要复杂些，但总体来说都是出现问题就去解决问题。当我们移除的是一个红色节点，那么根本就不会影响我们的性质4和性质5，我们不需要调整，但倘若我们移除的是一个黑色的节点，这时肯定会违反我们的性质5，所以我们只需要调整移除黑色节点的情况。分情况讨论下：

1. 如果兄弟节点是红色的，把兄弟节点染黑，父节点染红，左/右旋父节点；

2. 如果兄弟节点是黑色的，并且两个侄子节点都是黑色的，将兄弟节点染红，指针回溯至父亲节点；

3. 如果兄弟节点是黑色，的并且远侄子是黑色的，近侄子是红色的，将进侄子染黑，兄弟染红，左/右旋兄弟节点，进入下面情况 4 ；

4. 如果兄弟节点是黑色的，并且远侄子是红色的，近侄子随意，将兄弟节点染成父亲节点的颜色，父亲节点染黑，远侄子染黑，左/右旋父亲节点。

   

   ![img](https:////upload-images.jianshu.io/upload_images/4314397-a46ca001a1a1a959.png?imageMogr2/auto-orient/strip|imageView2/2/w/1151/format/webp)

   

```java
void solveLostBlack(TreeNode *pNode) {
        while (pNode != root && getColor(pNode) == black) {
            if (left(parent(pNode)) == pNode) {
                TreeNode *sib = brother(pNode);
                if (getColor(sib) == red) {
                    setColor(sib, black);
                    setColor(parent(pNode), red);
                    L_Rotation(parent(pNode));
                    sib = brother(pNode);
                }

                if (getColor(left(sib)) == black && getColor(right(sib)) == black) {
                    setColor(sib, red);
                    pNode = parent(pNode);
                } else {
                    if (getColor(right(sib)) == black) {
                        setColor(left(sib), black);
                        setColor(sib, red);
                        R_Rotation(sib);
                        sib = brother(pNode);
                    }

                    setColor(sib, getColor(parent(pNode)));
                    setColor(parent(pNode), black);
                    setColor(right(sib), black);
                    L_Rotation(parent(pNode));
                    pNode = root;
                }
            } else {
                TreeNode *sib = brother(pNode);
                if (getColor(sib) == red) {
                    setColor(sib, black);
                    setColor(parent(pNode), red);
                    R_Rotation(parent(pNode));
                    sib = brother(pNode);
                }

                if (getColor(left(sib)) == black && getColor(right(sib)) == black) {
                    setColor(sib, red);
                    pNode = parent(pNode);
                } else {
                    if (getColor(left(sib)) == black) {
                        setColor(right(sib), black);
                        setColor(sib, red);
                        L_Rotation(sib);
                        sib = brother(pNode);
                    }

                    setColor(sib, getColor(parent(pNode)));
                    setColor(parent(pNode), black);
                    setColor(left(sib), black);
                    R_Rotation(parent(pNode));
                    pNode = root;
                }
            }
        }
        pNode->color = black;
    }
```