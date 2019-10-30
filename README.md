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

### 并发面试题总结

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

```
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

