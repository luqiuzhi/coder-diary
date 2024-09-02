# JVM的运行时内存区域

之前讲到了JVM的内存模型，而JVM中的内存结构也是一个重要的知识点。

以JDK1.8为例：

![code-diary.drawio.svg](code-diary.drawio.svg){ thumbnail="true" }

我们一一对以上区域进行说明：

## 程序计数器(Program Counter Register)

程序计数器主要用于字节码解释器执行时，标记当前执行的字节码所在行。

字节码解释器通过程序计数器确定当前的线程执行到了class文件的哪一行，并以此来控制程序的顺序执行、分支、循环、跳转、异常处理等执行过程。

由于Java是多线程语言，每个线程的执行进度都不一样。在多线程的章节提到每个线程的CPU时间分片结束后就会切换到其他线程执行，在多线程环境中每个线程都需要记录好自己代码执行的进度，所以`程序计数器`
是线程私有的。

程序计数器中存储的是当前字节码指令的地址，是一块很小的内存区域，也是唯一一个不会有内存溢出(`OutOfMemoryError`)的区域。

> 为什么程序计数器区域不会有内存溢出？因为这是一个可预见大小的内存区域，无论什么情况下都不会超过最大值。在JVM的文档中，有如下描述：
>
> "The Java Virtual Machine's pc register is wide enough to hold a returnAddress or a native pointer on the specific
> platform."
>
{style="note"}

程序计数器的生命周期随线程的创建而创建，随线程的消亡而消亡。

## 虚拟机栈(VM Stack)

虚拟机栈是JVM运行时中的一块核心内存区域。由栈（stack）的特性可以知道，栈中的元素是**先进后出**的。

在Java程序运行过程中了，除了本地方法外，其余的方法间调用都要通过虚拟机栈来完成。和程序计数器一样，虚拟机栈也是线程私有的。

程序每进入一个方法，就往栈中压入（栈增加元素一般成为压入）一个栈帧，每跳出一个方法(执行完毕或return)就从栈中弹出（栈删除元素一般成为弹出）一个栈帧。

虚拟机栈由 ***栈帧*** 组成，每一个栈帧中都保存了局部变量表、操作栈、动态链接和返回地址。

![code-diary-虚拟机栈.drawio.svg](code-diary-虚拟机栈.drawio.svg) {thumbnail="true"}

### 局部变量表(Local Variable Table)

局部变量表其实最好理解，我们常说：局部变量是线程安全的。原因就在于局部变量是存储在局部变量表的，也就是在线程私有的虚拟机栈中，既然是线程私有的，自然也就是线程安全的了。

局部变量表中存储了当前方法定义的参数以及方法体中声明的局部变量。

局部变量表的最大容量在编译期就确定了，这是因为在局部变量表中存储是以下已知类型的数据：

- boolean
- byte
- short
- int
- long
- char
- float
- double
- reference
- ~~returnAddress~~

> 上面的`returnAddress`被删除的原因是从class编译版本号51开始，JVM的规范中就禁用了`jsr,jsr_w,ret`三个指令，所以`returnAddress`也就不常见到了。
>
> 关于这个问题在StackOverflow上还有一个小小讨论：<a href="https://stackoverflow.com/questions/57753497/what-does-at-returnaddress-mean-in-jvm">What does `at ReturnAddress` mean in JVM?</a> 

