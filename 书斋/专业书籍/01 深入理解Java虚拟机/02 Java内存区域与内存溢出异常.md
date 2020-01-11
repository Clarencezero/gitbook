# Java 内存区域与内存溢出异常

## **2.1 概述**

Java 与 C++之间有一堵由**内存分配**和**垃圾收集**技术所围成的高墙，墙外的人想进来，墙里面的人想出来。

## **2.2 Java 运行时数据区**

![Java 运行时数据区 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/jvm constructor1.png)

### 2.2.1 程序计数器

**程序计数器（Program Counter Register）:** 是一块较小的空间，用来指示当前线程所执行字节码的 <u> 行号 </u>。

**字节码解释器**: 工作时就是通过改变这个计数器的值来选定下一条需要执行的字节码指令，**分支、循环、跳转、异常处理、线程恢复** 等基础功能都需要依赖这个计数器来完成。

Java 虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现。在任一一个确定的时刻，一个处理器（对于多核处理器来说，就是一个内核）只会执行一条线程中的指令。因此，为了能够恢复线程切换之前的线程状态到正确位置，**每条线程都有一个独立的程序计数器**，各条线程之间互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

如果程正在执行的是一个**Java**方法，这个计数器记录的是正在执行虚拟机字节码指令的地址；如果正在执行的是**Native**方法，这个计数器则是为空（Undefined）。 此内存区域是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。

### 2.2.2 Java 虚拟机栈

![Java 虚拟机栈 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/jAVA.png)

- 线程私有, 生命周期与线程相同
- 描述 Java 方法执行的内存模型
  - 每个方法在执行的同时都会创建一个栈帧 (Stack Frame) 用于存储局部变量表、操作数栈、动态链接、方法出口等信息。
  - 当线程请求的栈深度超过最大值，会抛出 `StackOverflowError` 异常；
  - 如果线程请求的栈深度大于虚拟机所允许的深度, 将抛出`OutOfMemoryError`异常。      

> 局部变量表: 局部变量表中存放了**编译期**可知的基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（refrence 类型，可能是一个对象起始地址的应用指针、也可能是一个代表对象的句柄或者其它与此对象相关的地址）和 returnAddress（指向了一条字节码指令的地址）。
>
> 其中，64 位长度的 long 和 double 类型的数据会占用**2**个局部变量空间（Slot），其余数据类型会占用一个。局部变量所占用的空间在**编译期间**完成分配，**当进入一个方法时，这个方法在帧中所需要分配的局部变量的空间时完全可以确定的，在运行期间也不会改变这个空间的大小。**

### 2.2.3 本地方法栈

![本地方法栈 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/本地方法栈.png)

与虚拟机栈发挥的作用是非常相似的。区别是虚拟机栈为虚拟机执行 Java 方法 (字节码) 服务, 而本地方法栈则为虚拟机使用到的 native 方法服务。

HotSpot 直接就把本地方法栈的虚拟机栈合二为一。

### 2.2.4 Java 堆

Java 堆是 Java 虚拟机所管理的内存中最大的一块。在 java 虚拟机启动时创建。

几乎所有对象都在这里分配。但是随着 JIT 编译器的发展与逃逸分析技术逐渐成熟, **栈上分配、标题替换优化技术**将会导致一些微妙的变化发生。

堆分类:

- 粗略
  - 新生代
  - 老年代
- 细致
  - Eden 空间
  - From Survivor 空间
  - To Survivor 空间

![Java 堆与非堆简要 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/jvm heap and no-heap1.png)

无论如何划分, 都与存放的内容无关, 无论哪个区域, 存储的都仍然是对象实例,进一步划分的目的是为了更好的回收内存,或者更快地分配内存。

如果在堆中没有内存完成实例分配, 并且堆也无法再扩展时, 将会抛出`OutOfMemoryError`异常。

### 2.2.5 方法区

![方法区 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/方法区.jpg)

用于存储已被虚拟机加载的**类信息、常量池、静态变量、JIT 编译后的代码**等数据。

> 别名: Non-Heap(非堆)。永久代 (Permanent Generation)(只有 HotSpot 才有永久代概念)。Java8 取消永久代, 使用元空间代码。
>
> 元空间与永久代有着本质的区别。元空间并不存在虚拟机中, 而是使用本地内存。默认情况下, 元空间的大小受到本地内存限制, 但是可以通过以下参数来指定元空间的大小。
>
> ```shell
> -XX:MaxMetaspaceSize
> ```
>
> 更多有关内容可以参考[博客 ](https://www.cnblogs.com/paddix/p/5309550.html)

**Q:**为什么移除永久代

> - 它的大小是在启动时固定好的，很难进行调优。
> - HotSpot 内存类型也是 Java 对象，它可能会在 Full GC 中被移动，同时它对应用不透明，且是非强类型的，难以跟踪调试。
> - 简化 Full GC
> - 可以在 GC 不进行暂停的情况下并发地释放类数据。
>
> - ...

根据上面的各种原因，永久代最终被移除。方法区->Metaspace、字符串常量->Java Heap。
Metaspace 不再是存储在连续的堆空间上，而是移动到 metaspace 的`Native Memory(本地内存)`中。

### 2.2.6 运行时常量池

方法区的一部分。

Java 虚拟机规范没有对运行时常量池做任何细节的要求。

**Class 文件**除了有**类的版本、字段、方法、接口**等描述信息，还有一项信息就是常量池，用于存放编译时期生成的各种**字面量**和**符号引用**，这部分内容将在类加载后存放到方法区的运行时常量池。除了在编译期生成的常量，还允许动态生成，例如 String 类的 `intern()`。

当常量池无法申请到内存时就会抛出`OutOfMemoryError`异常。

## 2.3 HotSpot 虚拟机对象探秘

### 2.3.1 对象的创建

![HotSpot 对象创建 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/HotSpot%20create.png)



### 2.3.2 对象的内存布局

在 HotSpot 虚拟机中, 对象在内存中存储的布局可以分为 3 块区域: **对象头 (Header)、实例数据 (Instance Data) 和对象填充 (padding)。**

HotSpot 对象头包含两部分信息

- 用于存储对象自身的运行时数据, 如**哈希码 (HashCode)、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳**等。Mark Word 被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息。

![HotSpot mark word](https://cdn.jsdelivr.net/gh/Clarencezero/poi/32hotspotmark word.jpg)

- 类型指针, 即对象指向它的类元数据的指针, 虚拟机通过这个指针来确定这个对象是哪个类的实例。

实例数据部分, 对象真正存储的有效信息, 也是在程序代码中所定义的各种类型的字段内容。包含父类继承。存储顺序会受到虚拟机分配策略参数 (FieldsAllocationStyle) 和字段在 Java 源码中定义顺序的影响。



### 2.3.3 对象的访问定位

建立对象是为了使用对象, Java 程序需要通过栈上的 reference 数据来操作堆上的具体对象。

reference 类型包含三种: 

- class types
- array types
- interface types

reference 在虚拟机规范中只规定了一个指向对象的引用, 并没有规定方式。目前主流虚拟机使用**句柄**和**直接指针**两种。



- 句柄

  Java 堆中将会划分出一块内存来作为句柄池, reference 中存储的就是对象的句柄地址。
  
  最大好处就是 reference 中存储的是稳定的句柄地址, 在对象被移动时只会改变句柄中的实例数据指针, 而 reference 本身不需要修改。

![通过句柄访问对象 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/jubing.png)

- 直接指针

  速度更快。节省了一次指针定位的时间开销。HotSpot 默认使用该种对象访问方式。

![直接指针 ](https://cdn.jsdelivr.net/gh/Clarencezero/poi/get object direct2.png)



## 参考文章

[锁、虚拟机 ](http://www.liuhaihua.cn/archives/647554.html)

[The Java Virtual Machine Specification, Java SE 8 Edition](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)







