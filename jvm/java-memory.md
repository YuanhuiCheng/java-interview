# Java 内存区域

### 运行时数据区域

JDK 1.8 之后：

![运行时数据区域](../img/java-memory.png)

__线程私有__：
- 程序计数器
- 虚拟机栈
- 本地方法栈

__线程共享__：
- 堆
- 方法区
- 直接内存 （非运行时数据区的一部分）

### 区域详解

#### 程序计数器

- __字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令__。分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完成。
- 为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。
- 程序计数器是唯一一个不会出现 `OutOfMemoryError` 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡。

#### Java 虚拟机栈

- 线程私有，生命周期和线程相同，描述的是 Java 方法执行的内存模型。
- 由一个个栈帧组成，每个栈帧中拥有：局部变量表，操作数栈，动态链接，方法出口信息...
    - 局部变量表主要存放了编译期可知的各种数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference 类型，它不同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或其他与此对象相关的位置）。
- 会出现两种错误：`StackOverFlowError` 和 `OutOfMemoryError`。
- Java 栈可以类比数据结构中栈，Java 栈中保存的主要内容是栈帧，每一次函数调用都会有一个对应的栈帧被压入 Java 栈，每一个函数调用结束后 （return 或者抛出异常），都会有一个栈帧被弹出。

#### 本地方法栈

和虚拟机栈所发挥的作用非常相似，区别是： 虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

#### 堆

- 此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。
    - 从 JDK 1.7 开始已经默认开启逃逸分析，如果某些方法中的对象引用没有被返回或者未被外面使用（也就是未逃逸出去），那么对象可以直接在栈上分配内存。
- Java 堆是垃圾收集器管理的主要区域，因此也被称作 `GC堆（Garbage Collected Heap）`。从垃圾回收的角度，由于现在收集器基本都采用分代垃圾收集算法，所以 Java 堆还可以细分为：新生代和老年代；再细致一点有：Eden、Survivor、Old 等空间。进一步划分的目的是更好地回收内存，或者更快地分配内存。
- 堆里最容易出现 `OutOfMemoryError`，比如：
    - `java.lang.OutOfMemoryError: GC Overhead Limit Exceeded`： 当 JVM 花太多时间执行垃圾回收并且只能回收很少的堆空间时，就会发生此错误。
    - `java.lang.OutOfMemoryError: Java heap space`: 假如在创建新的对象时, 堆内存中的空间不足以存放新创建的对象, 就会引发此错误。(和配置的最大堆内存有关，且受制于物理内存大小。最大堆内存可通过-Xmx参数配置，若没有特别配置，将会使用默认值。

- 在 JDK 7 版本及 JDK 7 版本之前，堆内存被通常分为下面三部分：

    - 新生代内存 (Young Generation) 
    - 老生代 (Old Generation) 
    - 永久代(Permanent Generation)

    JDK 8 版本之后 PermGen 已被 Metaspace (元空间) 取代，元空间使用的是直接内存。

#### 方法区

- __方法区与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。__


为什么要将永久代 (PermGen) 替换为元空间 (MetaSpace) 呢?

- 整个永久代有一个 JVM 本身设置的固定大小上限，无法进行调整，而元空间使用的是直接内存，受本机可用内存的限制，虽然元空间仍旧可能溢出，但是比原来出现的几率会更小。
- 元空间里面存放的是类的元数据，这样加载多少类的元数据就不由 MaxPermSize 控制了, 而由系统的实际可用空间来控制，这样能加载的类就更多了。

#### 运行时常量池

运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池表（用于存放编译期生成的各种字面量和符号引用） 既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 错误。

> JDK1.7 之前运行时常量池逻辑包含字符串常量池存放在方法区, 此时 hotspot 虚拟机对方法区的实现为永久代。
JDK1.7 字符串常量池被从方法区拿到了堆中, 这里没有提到运行时常量池,也就是说字符串常量池被单独拿到堆,运行时常量池剩下的东西还在方法区, 也就是 hotspot 中的永久代。
JDK1.8 hotspot 移除了永久代用元空间(Metaspace)取而代之, 这时候字符串常量池还在堆, 运行时常量池还在方法区, 只不过方法区的实现从永久代变成了元空间(Metaspace)。

#### 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 错误出现。

本机直接内存的分配不会受到 Java 堆的限制，但是，既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。