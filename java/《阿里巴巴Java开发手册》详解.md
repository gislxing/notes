# 《阿里巴巴Java开发手册》详解

## 编码

### Integer缓存问题分析

#### 前言

《手册》第 7 页有一段关于包装对象之间值的比较问题的规约 [1](https://www.imooc.com/read/55/article/1139#fn1)：

> 【强制】所有整型包装类对象之间值的比较，全部使用 equals 方法比较。
> 说明：对于 Integer var = ? 在 - 128 至 127 范围内的赋值，Integer 对象是在 IntegerCache.cache 产 生，会复用已有对象，这个区间内的 Integer 值可以直接使用 == 进行判断，但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象，这是一个大坑，推荐使用 equals 方法进行判断。

这条建议非常值得大家关注， 而且该问题在 Java 面试中十分常见。

我们还需要思考以下几个问题：

- 如果不看《手册》，我们如何知道 `Integer var = ?` 会缓存 -128 到 127 之间的赋值？
- 为什么会缓存这个范围的赋值？
- 我们如何学习和分析类似的问题？



#### Integer 缓存问题分析

我们先看下面的示例代码，并思考该段代码的输出结果：

```java
public class IntTest {
	public static void main(String[] args) {
	    Integer a = 100, b = 100, c = 150, d = 150;
	    System.out.println(a == b);
	    System.out.println(c == d);
	}
}
```

通过运行代码可以得到答案，程序输出的结果分别为： `true` , `false`。

**那么为什么答案是这样？**

结合《手册》的描述很多人可能会颇有自信地回答：**因为缓存了 -128 到 127 之间的数值**，就没有然后了。

那么为什么会缓存这一段区间的数值？缓存的区间可以修改吗？其它的包装类型有没有类似缓存？

**what? 咋还有这么多问题？这谁知道啊**！

莫急，且看下面的分析。

#### 源码分析法

首先我们可以通过源码对该问题进行分析。

我们知道，`Integer var = ?` 形式声明变量，会通过 `java.lang.Integer#valueOf(int)` 来构造 `Integer` 对象。

很多人可能会说：“你咋能知道这个呢”？

如果不信大家可以打断点，运行程序后会调到这里，总该信了吧？（后面还会再作解释）。

我们先看该函数源码：

```java
/**
 * Returns an {@code Integer} instance representing the specified
 * {@code int} value.  If a new {@code Integer} instance is not
 * required, this method should generally be used in preference to
 * the constructor {@link #Integer(int)}, as this method is likely
 * to yield significantly better space and time performance by
 * caching frequently requested values.
 *
 * This method will always cache values in the range -128 to 127,
 * inclusive, and may cache other values outside of this range.
 *
 * @param  i an {@code int} value.
 * @return an {@code Integer} instance representing {@code i}.
 * @since  1.5
 */
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

通过源码可以看出，如果用 `Ineger.valueOf(int)` 来创建整数对象，参数大于等于整数缓存的最小值（ `IntegerCache.low` ）并小于等于整数缓存的最大值（ `IntegerCache.high`）, 会直接从缓存数组 (`java.lang.Integer.IntegerCache#cache`) 中提取整数对象；否则会 `new` 一个整数对象。

**那么这里的缓存最大和最小值分别是多少呢？**

从上述注释中我们可以看出，最小值是 -128, 最大值是 127。

**那么为什么会缓存这一段区间的整数对象呢？**

通过注释我们可以得知：**如果不要求必须新建一个整型对象，缓存最常用的值（提前构造缓存范围内的整型对象），会更省空间，速度也更快。**

这给我们一个非常重要的启发：

> 如果想减少内存占用，提高程序运行的效率，可以将常用的对象提前缓存起来，需要时直接从缓存中提取。

那么我们再思考下一个问题： **`Integer` 缓存的区间可以修改吗？**

通过上述源码和注释我们还无法回答这个问题，接下来，我们继续看 `java.lang.Integer.IntegerCache` 的源码：

```java
/**
 * Cache to support the object identity semantics of autoboxing for values between
 * -128 and 127 (inclusive) as required by JLS.
 *
 * The cache is initialized on first usage.  The size of the cache
 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
 * During VM initialization, java.lang.Integer.IntegerCache.high property
 * may be set and saved in the private system properties in the
 * sun.misc.VM class.
 */

private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];
    static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
           // 省略其它代码
    }
      // 省略其它代码
}
```

通过 `IntegerCache` 代码和注释我们可以看到，最小值是固定值 -128， 最大值并不是固定值，缓存的最大值是可以通过虚拟机参数 `-XX:AutoBoxCacheMax=}` 或 `-Djava.lang.Integer.IntegerCache.high=` 来设置的，未指定则为 127。

因此可以通过修改这两个参数其中之一，让缓存的最大值大于等于 150。

如果作出这种修改，示例的输出结果便会是： `true`,`true`。

**学到这里是不是发现，对此问题的理解和最初的想法有些不同呢？**

这段注释也解答了为什么要缓存这个范围的数据：

> **是为了自动装箱时可以复用这些对象** ，这也是 JLS[2](https://www.imooc.com/read/55/article/1139#fn2) 的要求。

我们可以参考 JLS 的 [Boxing Conversion 部分](https://docs.oracle.com/javase/specs/jls/se8/html/jls-5.html#jls-5.1.7)的相关描述。

> If the value`p`being boxed is an integer literal of type `int`between `-128`and `127`inclusive ([§3.10.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.1)), or the boolean literal `true`or`false`([§3.10.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.3)), or a character literal between `'\u0000'`and `'\u007f'`inclusive ([§3.10.4](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.4)), then let `a`and `b`be the results of any two boxing conversions of `p`. It is always the case that `a`==`b`.
>
> 在 -128 到 127 （含）之间的 int 类型的值，或者 boolean 类型的 true 或 false， 以及范围在’\u0000’和’\u007f’ （含）之间的 char 类型的数值 p， 自动包装成 a 和 b 两个对象时， 可以使用 a == b 判断 a 和 b 的值是否相等。

#### 反汇编法

那么究竟 `Integer var = ?` 形式声明变量，是不是通过 `java.lang.Integer#valueOf(int)` 来构造 `Integer` 对象呢？ 总不能都是猜测 N 个可能的函数，然后断点调试吧？

**如果遇到其它类似的问题，没人告诉我底层调用了哪个方法，该怎么办？** 囧…

这类问题有个杀手锏，可以通过对编译后的 class 文件进行反汇编来查看。

首先编译源代码：`javac IntTest.java`

然后需要对代码进行反汇编，执行：`javap -c IntTest`

> 如果想了解 `javap` 的用法，直接输入 `javap -help` 查看用法提示（很多命令行工具都支持 `-help` 或 `--help` 给出用法提示）。
> ![图片描述](img/5db654e80001823606050363.png)

反编译后，我们得到以下代码：

```java
Compiled from "IntTest.java"
public class com.chujianyun.common.int_test.IntTest {
  public com.chujianyun.common.int_test.IntTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        100
       2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       5: astore_1
       6: bipush        100
       8: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      11: astore_2
      12: sipush        150
      15: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      18: astore_3
      19: sipush        150
      22: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      25: astore        4
      27: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      30: aload_1
      31: aload_2
      32: if_acmpne     39
      35: iconst_1
      36: goto          40
      39: iconst_0
      40: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
      43: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      46: aload_3
      47: aload         4
      49: if_acmpne     56
      52: iconst_1
      53: goto          57
      56: iconst_0
      57: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
      60: return
}
```

可以明确得 "看到" 这四个 ``Integer var = ? `形式声明的变量的确是通过` java.lang.Integer#valueOf(int) `来构造` Integer` 对象的。

**接下来对汇编后的代码进行详细分析，如果看不懂可略过**：

根据[《Java Virtual Machine Specification : Java SE 8 Edition》](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)[3](https://www.imooc.com/read/55/article/1139#fn3)，后缩写为 JVMS , 第 6 章 [虚拟机指令集](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)的相关描述以及《深入理解 Java 虚拟机》[4](https://www.imooc.com/read/55/article/1139#fn4) 414-149 页的 附录 B “虚拟机字节码指令表”。 我们对上述指令进行解读：

偏移为 0 的指令为：`bipush 100` ，其含义是将单字节整型常量 100 推入操作数栈的栈顶；

偏移为 2 的指令为：`invokestatic #2 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;` 表示调用一个 `static` 函数，即 `java.lang.Integer#valueOf(int)`；

偏移为 5 的指令为：`astore_1` ，其含义是从操作数栈中弹出对象引用，然后将其存到第 1 个局部变量 Slot 中；

偏移 6 到 25 的指令和上面类似；

偏移为 30 的指令为 `aload_1` ，其含义是从第 1 个局部变量 Slot 取出对象引用（即 a），并将其压入栈；

偏移为 31 的指令为 `aload_2` ，其含义是从第 2 个局部变量 Slot 取出对象引用（即 b），并将其压入栈；

偏移为 32 的指令为 `if_acmpn`，该指令为条件跳转指令，`if_` 后以 a 开头表示对象的引用比较。

由于该指令有以下特性：

> - `if_acmpeq` 比较栈两个引用类型数值，相等则跳转
> - `if_acmpne` 比较栈两个引用类型数值，不相等则跳转

由于 `Integer` 的缓存问题，所以 a 和 b 引用指向同一个地址，因此此条件不成立（成立则跳转到偏移为 39 的指令处），执行偏移为 35 的指令。

偏移为 35 的指令: `iconst_1`，其含义为将常量 1 压栈（ Java 虚拟机中 boolean 类型的运算类型为 int ，其中 true 用 1 表示，详见 [2.11.1 数据类型和 Java 虚拟机](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.1)。

然后执行偏移为 36 的 `goto` 指令，跳转到偏移为 40 的指令。

偏移为 40 的指令：`invokevirtual #4 // Method java/io/PrintStream.println:(Z)V`。

可知参数描述符为 `Z` ，返回值描述符为 `V`。

根据 [4.3.2 字段描述符](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.2) ，可知 `FieldType` 的字符为 `Z` 表示 `boolean` 类型， 值为 `true` 或 `false`。
根据 [4.3.3 字段描述符](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.3.3) ，可知返回值为 `void`。

因此可以知，最终调用了 `java.io.PrintStream#println(boolean)` 函数打印栈顶常量即 `true`。

然后比较执行偏移 43 到 57 之间的指令，比较 c 和 d， 打印 `false` 。

执行偏移为 60 的指令，即 `retrun` ，程序结束。

可能有些朋友会对反汇编的代码有些抵触和恐惧，这都是非常正常的现象。

我们分析和研究问题的时候，**看懂核心逻辑即可**，不要纠结于细节，而失去了重点。

一回生两回熟，随着遇到的例子越来越多，遇到类似的问题时，会喜欢上 `javap` 来分析和解决问题。

如果想深入学习 java 反汇编，强烈建议结合官方的 JVMS 或其中文版:《Java 虚拟机规范》这本书进行拓展学习。

如果大家不喜欢命令行的方式进行 Java 的反汇编，这里推荐一个简单易用的可视化工具：[classpy](https://github.com/zxh0/classpy) ，大家可以自行了解学习。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

#### Long 的缓存问题分析

我们学习的目的之一就是要学会举一反三。因此我们对 `Long` 也进行类似的研究，探究两者之间有何异同。



#### 源码分析

类似的，我们接下来分析 `java.lang.Long#valueOf(long)` 的源码：

```java
/**
 * Returns a {@code Long} instance representing the specified
 * {@code long} value.
 * If a new {@code Long} instance is not required, this method
 * should generally be used in preference to the constructor
 * {@link #Long(long)}, as this method is likely to yield
 * significantly better space and time performance by caching
 * frequently requested values.
 *
 * Note that unlike the {@linkplain Integer#valueOf(int)
 * corresponding method} in the {@code Integer} class, this method
 * is <em>not</em> required to cache values within a particular
 * range.
 *
 * @param  l a long value.
 * @return a {@code Long} instance representing {@code l}.
 * @since  1.5
 */
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

发现该函数的写法和 `Ineger.valueOf(int)` 非常相似。

我们同样也看到， `Long` 也用到了缓存。 使用 `java.lang.Long#valueOf(long)` 构造 `Long` 对象时，值在 **[-128, 127]** 之间的 `Long` 对象直接从缓存对象数组中提取。

而且注释同样也提到了：**缓存的目的是为了提高性能**。

但是通过注释我们发现这么一段提示：

> Note that unlike the {@linkplain Integer#valueOf(int) corresponding method} in the {@code Integer} class, this method is *not* required to cache values within a particular range.
>
> 注意：和 `Ineger.valueOf(int)` 不同的是，此方法并没有被要求缓存特定范围的值。

这也正是上面源码中缓存范围判断的注释为何用 `// will cache` 的原因（可以对比一下上面 `Integer` 的缓存的注释）。

因此我们可知，虽然此处采用了缓存，但应该不是 JLS 的要求。

**那么 `Long` 类型的缓存是如何构造的呢？**

我们查看缓存数组的构造：

```java
private static class LongCache {
    private LongCache(){}

    static final Long cache[] = new Long[-(-128) + 127 + 1];

    static {
        for(int i = 0; i < cache.length; i++)
            cache[i] = new Long(i - 128);
    }
}
```

可以看到，它是在静态代码块中填充缓存数组的。

#### 反编译

同样地我们也编写一个示例片段：

```java
public class LongTest {

    public static void main(String[] args) {
        Long a = -128L, b = -128L, c = 150L, d = 150L;
        System.out.println(a == b);
        System.out.println(c == d);
    }
}
```

编译源代码： `javac LongTest.java`

对编译后的类文件进行反汇编: `javap -c LongTest`

得到下面反编译的代码：

```java
public class com.imooc.basic.learn_int.LongTest {
  public com.imooc.basic.learn_int.LongTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc2_w        #2                  // long -128l
       3: invokestatic  #4                  // Method java/lang/Long.valueOf:(J)Ljava/lang/Long;
       6: astore_1
       7: ldc2_w        #2                  // long -128l
      10: invokestatic  #4                  // Method java/lang/Long.valueOf:(J)Ljava/lang/Long;
      13: astore_2
      14: ldc2_w        #5                  // long 150l
      17: invokestatic  #4                  // Method java/lang/Long.valueOf:(J)Ljava/lang/Long;
      20: astore_3
      21: ldc2_w        #5                  // long 150l
      24: invokestatic  #4                  // Method java/lang/Long.valueOf:(J)Ljava/lang/Long;
      27: astore        4
      29: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      32: aload_1
      33: aload_2
      34: if_acmpne     41
      37: iconst_1
      38: goto          42
      41: iconst_0
      42: invokevirtual #8                  // Method java/io/PrintStream.println:(Z)V
      45: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
      48: aload_3
      49: aload         4
      51: if_acmpne     58
      54: iconst_1
      55: goto          59
      58: iconst_0
      59: invokevirtual #8                  // Method java/io/PrintStream.println:(Z)V
      62: return
}
```

我们从上述代码中发现 `Long var = ?` 的确是通过 `java.lang.Long#valueOf(long)` 来构造对象的。



#### 总结

本小节通过源码分析法、阅读 JLS 和 JVMS、使用反汇编法，对 `Integer` 和 `Long` 缓存的目的和实现方式问题进行了深入分析。

让大家能够通过更丰富的手段来学习知识和分析问题，通过对缓存目的的思考来学到更通用和本质的东西。

本节使用的几种手段将是我们未来常用的方法，也是工作进阶的必备技能和一个程序员专业程度的体现，希望大家未来能够多动手实践。

下一节我们将介绍 Java 序列化相关问题，包括序列化的定义，序列化常见的方案，序列化的坑点等。



#### 课后题

**第 1 题**：请大家根据今天的研究分析过程，对下面的一个示例代码进行分析。

```java
public class CharacterTest {
    public static void main(String[] args) {
        Character a = 126, b = 126, c = 128, d = 128;
        System.out.println(a == b);
        System.out.println(c == d);
    }
}
```

**第 2 题**： 结合今天的讲解，请自行对 `Character`、 `Short` 、`Boolean` 的缓存问题进行分析，并比较它们的异同。

### Java序列化引发的血案

#### 前言

《手册》第 9 页 “OOP 规约” 部分有一段关于序列化的约定 [1](https://www.imooc.com/read/55/article/1140#fn1)：

> 【强制】当序列化类新增属性时，请不要修改 serialVersionUID 字段，以避免反序列失败；如果完全不兼容升级，避免反序列化混乱，那么请修改 serialVersionUID 值。
> 说明：注意 serialVersionUID 值不一致会抛出序列化运行时异常。

我们应该思考下面几个问题：

- 序列化和反序列化到底是什么？
- 它的主要使用场景有哪些？
- Java 序列化常见的方案有哪些？
- 各种常见序列化方案的区别有哪些？
- 实际的业务开发中有哪些坑点？

接下来将从这几个角度去研究这个问题。



#### 序列化和反序列化是什么？为什么需要它？

**序列化**是将内存中的对象信息转化成可以存储或者传输的数据到临时或永久存储的过程。而**反序列化**正好相反，是从临时或永久存储中读取序列化的数据并转化成内存对象的过程。

![图片描述](img/5db655fc000168f810200424.png)

**那么为什么需要序列化和反序列化呢？**

希望大家能够养成从本源上思考这个问题的思维方式，即思考它为什么会出现，而不是单纯记忆。

> 大家可以回忆一下，平时都是如果将文字文件、图片文件、视频文件、软件安装包等传给小伙伴时，这些资源在计算机中存储的方式是怎样的。
>
> 进而再思考，Java 中的对象如果需要存储或者传输应该通过什么形式呢？

我们都知道，一个文件通常是一个 m 个字节的序列：B0, B1, …, Bk, …, Bm-1。所有的 I/O 设备（例如网络、磁盘和终端）都被模型化为文件，而所有的输入和输出都被当作对应文件的读和写来执行。[2](https://www.imooc.com/read/55/article/1140#fn2)

因此本质上讲，文本文件，图片、视频和安装包等文件底层都被转化为二进制字节流来传输的，对方得文件就需要对文件进行解析，因此就需要有能够根据不同的文件类型来解码出文件的内容的程序。

大家试想一个典型的场景：如果要实现 Java 远程方法调用，就需要将调用结果通过网路传输给调用方，如果调用方和服务提供方不在一台机器上就很难共享内存，就需要将 Java 对象进行传输。而想要将 Java 中的对象进行网络传输或存储到文件中，就需要将对象转化为二进制字节流，这就是所谓的序列化。存储或传输之后必然就需要将二进制流读取并解析成 Java 对象，这就是所谓的反序列化。

序列化的主要目的是：**方便存储到文件系统、数据库系统或网络传输等**。

实际开发中常用到序列化和反序列化的场景有：

- 远程方法调用（RPC）的框架里会用到序列化。
- 将对象存储到文件中时，需要用到序列化。
- 将对象存储到缓存数据库（如 Redis）时需要用到序列化。
- 通过序列化和反序列化的方式实现对象的深拷贝。



#### 常见的序列化方式

常见的序列化方式包括 Java 原生序列化、Hessian 序列化、Kryo 序列化、JSON 序列化等。



##### Java 原生序列化

正如前面章节讲到的，对于 JDK 中有的类，最好的学习方式之一就是直接看其源码。

`Serializable` 的源码非常简单，只有声明，没有属性和方法：

```java
// 注释太长，省略
public interface Serializable {
}
```

在学习源码注释之前，希望大家可以站在设计者的角度，先思考一个问题：如果一个类序列化到文件之后，类的结构发生变化还能否保证正确地反序列化呢？

答案显然是不确定的。

**那么如何判断文件被修改过了呢？** 通常可以通过加密算法对其进行签名，文件作出任何修改签名就会不一致。但是 Java 序列化的场景并不适合使用上述的方案，因为类文件的某些位置加个空格，换行等符号类的结构没有发生变化，这个签名就不应该发生变化。还有一个类新增一个属性，之前的属性都是有值的，之前都被序列化到对象文件中，有些场景下还希望反序列化时可以正常解析，怎么办呢？

那么是否可以通过约定一个唯一的 ID，通过 ID 对比，不一致就认为不可反序列化呢？

**实现序列化接口后，如果开发者不手动指定该版本号 ID 怎么办？**

既然 Java 序列化场景下的 “签名” 应该根据类的特点生成，我们是否可以不指定序列化版本号就默认根据类名、属性和函数等计算呢？

如果针对某个自己定义的类，想自定义序列化和反序列化机制该如何实现呢？支持吗？

带着这些问题我们继续看序列化接口的注释。

`Serializable` 的源码注释特别长，其核心大致作了下面的说明：

Java 原生序列化需要实现 `Serializable` 接口。序列化接口不包含任何方法和属性等，它只起到序列化标识作用。

一个类实现序列化接口则其子类型也会继承序列化能力，但是实现序列化接口的类中有其他对象的引用，则其他对象也要实现序列化接口。序列化时如果抛出 `NotSerializableException` 异常，说明该对象没有实现 `Serializable` 接口。

每个序列化类都有一个叫 `serialVersionUID` 的版本号，反序列化时会校验待反射的类的序列化版本号和加载的序列化字节流中的版本号是否一致，如果序列化号不一致则会抛出 `InvalidClassException` 异常。

强烈推荐每个序列化类都手动指定其 `serialVersionUID`，如果不手动指定，那么编译器会动态生成默认的序列化号，因为这个默认的序列化号和类的特征以及编译器的实现都有关系，很容易在反序列化时抛出 `InvalidClassException` 异常。建议将这个序列化版本号声明为私有，以避免运行时被修改。

实现序列化接口的类可以提供自定义的函数修改默认的序列化和反序列化行为。

自定义序列化方法：

```java
private void writeObject(ObjectOutputStream out) throws IOException;
```

自定义反序列化方法：

```java
private void readObject(ObjectInputStream in) 
  throws IOException, ClassNotFoundException;
```

通过自定义这两个函数，可以实现序列化和反序列化不可序列化的属性，也可以对序列化的数据进行数据的加密和解密处理。



##### Hessian 序列化

Hessian 是一个动态类型，二进制序列化，也是一个基于对象传输的网络协议。Hessian 是一种跨语言的序列化方案，序列化后的字节数更少，效率更高。Hessian 序列化会把复杂对象的属性映射到 `Map` 中再进行序列化。



##### Kryo 序列化

Kryo 是一个快速高效的 Java 序列化和克隆工具。Kryo 的目标是快速、字节少和易用。Kryo 还可以自动进行深拷贝或者浅拷贝。Kryo 的拷贝是对象到对象的拷贝而不是对象到字节，再从字节到对象的恢复。Kryo 为了保证序列化的高效率，会提前加载需要的类，这会带一些消耗，但是这是序列化后文件较小且反序列化非常快的重要原因。



##### JSON 序列化

JSON (JavaScript Object Notation) 是一种轻量级的数据交换方式。JSON 序列化是基于 JSON 这种结构来实现的。JSON 序列化将对象转化成 JSON 字符串，JSON 反序列化则是将 JSON 字符串转回对象的过程。常用的 JSON 序列化和反序列化的库有 Jackson、GSON、Fastjson 等。



#### Java 常见的序列化方案对比

我们想要对比各种序列化方案的优劣无外乎两点，一点是查资料，一点是自己写代码验证。



##### Java 原生序列化

Java 序列化的优点是：对对象的结构描述清晰，反序列化更安全。主要缺点是：效率低，序列化后的二进制流较大。



##### Hessian 序列化

Hession 序列化二进制流较 Java 序列化更小，且序列化和反序列化耗时更短。但是父类和子类有相同类型属性时，由于先序列化子类再序列化父类，因此反序列化时子类的同名属性会被父类的值覆盖掉，开发时要特别注意这种情况。

> Hession2.0 序列化二进制流大小是 Java 序列化的 50%，序列化耗时是 Java 序列化的 30%，反序列化的耗时是 Java 序列化的 20%。 [3](https://www.imooc.com/read/55/article/1140#fn3)

编写待测试的类：

```java
@Data
public class PersonHessian implements Serializable {
    private Long id;
    private String name;
    private Boolean male;
}

@Data
public class Male extends PersonHessian {
    private Long id;
}
```

编写单测来模拟序列化继承覆盖问题：

```java
/**
 * 验证Hessian序列化继承覆盖问题
 */
@Test
public void testHessianSerial() throws IOException {
    HessianSerialUtil.writeObject(file, male);
    Male maleGet = HessianSerialUtil.readObject(file);
    // 相等
    Assert.assertEquals(male.getName(), maleGet.getName());
    // male.getId()结果是1，maleGet.getId()结果是null
    Assert.assertNull(maleGet.getId());
    Assert.assertNotEquals(male.getId(), maleGet);
}
```

上述单测示例验证了：反序列化时子类的同名属性会被父类的值覆盖掉的问题。



##### Kryo 序列化

Kryo 优点是：速度快、序列化后二进制流体积小、反序列化超快。但是缺点是：跨语言支持复杂。注册模式序列化更快，但是编程更加复杂。



##### JSON 序列化

JSON 序列化的优势在于可读性更强。主要缺点是：没有携带类型信息，只有提供了准确的类型信息才能准确地进行反序列化，这点也特别容易引发线上问题。

下面给出使用 Gson 框架模拟 JSON 序列化时遇到的反序列化问题的示例代码：

```java
/**
 * 验证GSON序列化类型错误
 */
@Test
public void testGSON() {
    Map<String, Object> map = new HashMap<>();
    final String name = "name";
    final String id = "id";
    map.put(name, "张三");
    map.put(id, 20L);

    String jsonString = GSONSerialUtil.getJsonString(map);
    Map<String, Object> mapGSON = GSONSerialUtil.parseJson(jsonString, Map.class);
    // 正确
    Assert.assertEquals(map.get(name), mapGSON.get(name));
    // 不等  map.get(id)为Long类型 mapGSON.get(id)为Double类型
    Assert.assertNotEquals(map.get(id).getClass(), mapGSON.get(id).getClass());
    Assert.assertNotEquals(map.get(id), mapGSON.get(id));
}
```

下面给出使用 fastjson 模拟 JSON 反序列化问题的示例代码：

```java
/**
 * 验证FatJson序列化类型错误
 */
@Test
public void testFastJson() {
    Map<String, Object> map = new HashMap<>();
    final String name = "name";
    final String id = "id";
    map.put(name, "张三");
    map.put(id, 20L);

    String fastJsonString = FastJsonUtil.getJsonString(map);
    Map<String, Object> mapFastJson = FastJsonUtil.parseJson(fastJsonString, Map.class);

    // 正确
    Assert.assertEquals(map.get(name), mapFastJson.get(name));
    // 错误  map.get(id)为Long类型 mapFastJson.get(id)为Integer类型
    Assert.assertNotEquals(map.get(id).getClass(), mapFastJson.get(id).getClass());
    Assert.assertNotEquals(map.get(id), mapFastJson.get(id));
}
```

大家还可以通过单元测试构造大量复杂对象对比各种序列化方式或框架的效率。

如定义下列测试类为 User，包括以下多种类型的属性：

```java
@Data
public class User implements Serializable {
    private Long id;
    private String name;
    private Integer age;
    private Boolean sex;
    private String nickName;
    private Date birthDay;
    private Double salary;
}
```



##### 各种常见的序列化性能排序

实验的版本：kryo-shaded 使用 4.0.2 版本，gson 使用 2.8.5 版本，hessian 用 4.0.62 版本。

实验的数据：构造 50 万 User 对象运行多次。

大致得出一个结论：

- 从二进制流大小来讲：JSON 序列化 > Java 序列化 > Hessian2 序列化 > Kryo 序列化 > Kryo 序列化注册模式；
- 从序列化耗时而言来讲：GSON 序列化 > Java 序列化 > Kryo 序列化 > Hessian2 序列化 > Kryo 序列化注册模式；
- 从反序列化耗时而言来讲：GSON 序列化 > Java 序列化 > Hessian2 序列化 > Kryo 序列化注册模式 > Kryo 序列化；
- 从总耗时而言：Kryo 序列化注册模式耗时最短。

> 注：由于所用的序列化框架版本不同，对象的复杂程度不同，环境和计算机性能差异等原因结果可能会有出入。



#### 序列化引发的一个血案

接下来我们看下面的一个案例：

> 前端调用服务 A，服务 A 调用服务 B，服务 B 首次接到请求会查 DB，然后缓存到 Redis（缓存 1 个小时）。服务 A 根据服务 B 返回的数据后执行一些处理逻辑，处理后形成新的对象存到 Redis（缓存 2 个小时）。
>
> 服务 A 通过 Dubbo 来调用服务 B，A 和 B 之间数据通过 `Map` 类型传输，服务 B 使用 Fastjson 来实现 JSON 的序列化和反序列化。
>
> 服务 B 的接口返回的 `Map` 值中存在一个 `Long` 类型的 `id` 字段，服务 A 获取到 `Map` ，取出 `id` 字段并强转为 `Long` 类型使用。

执行的流程如下：
![图片描述](img/5db656b00001507517500718.png)通过分析我们发现，服务 A 和服务 B 的 RPC 调用使用 Java 序列化，因此类型信息不会丢失。

但是由于服务 B 采用 JSON 序列化进行缓存，第一次访问没啥问题，其执行流程如下：
![图片描述](img/5db656cb000199cf08570550.png)

**如果服务 A 开启了缓存**，服务 A 在第一次请求服务 B 后，缓存了运算结果，且服务 A 缓存时间比服务 B 长，因此不会出现错误。
![图片描述](img/5db656ee0001020406490340.png)
**如果服务 A 不开启缓存**，服务 A 会请求服务 B ，由于首次请求时，服务 B 已经缓存了数据，服务 B 从 Redis（B）中反序列化得到 `Map`。流程如下图所示：

![图片描述](img/5db657060001151308390423.png)
然而问题来了： 服务 A 从 Map 取出此 `Id` 字段，强转为 `Long` 时会出现类型转换异常。

最后定位到原因是 Json 反序列化 Map 时如果原始值小于 Int 最大值，反序列化后原本为 Long 类型的字段，变为了 Integer 类型，服务 B 的同学紧急修复。

服务 A 开启缓存时， 虽然采用了 JSON 序列化存入缓存，但是采用 DTO 对象而不是 Map 来存放属性，所以 JSON 反序列化没有问题。

**因此大家使用二方或者三方服务时，当对方返回的是 `Map` 类型的数据时要特别注意这个问题**。

> 作为服务提供方，可以采用 JDK 或者 Hessian 等序列化方式；
>
> 作为服务的使用方，我们不要从 Map 中一个字段一个字段获取和转换，可以使用 JSON 库直接将 Map 映射成所需的对象，这样做不仅代码更简洁还可以避免强转失败。

代码示例：

```java
@Test
public void testFastJsonObject() {
    Map<String, Object> map = new HashMap<>();
    final String name = "name";
    final String id = "id";
    map.put(name, "张三");
    map.put(id, 20L);

    String fastJsonString = FastJsonUtil.getJsonString(map);
    // 模拟拿到服务B的数据
    Map<String, Object> mapFastJson = FastJsonUtil.parseJson(fastJsonString,map.getClass());
    // 转成强类型属性的对象而不是使用map 单个取值
    User user = new JSONObject(mapFastJson).toJavaObject(User.class);
    // 正确
    Assert.assertEquals(map.get(name), user.getName());
    // 正确
    Assert.assertEquals(map.get(id), user.getId());
}
```



#### 总结

本节的主要讲解了序列化的主要概念、主要实现方式，以及序列化和反序列化的几个坑点，希望大家在实际业务开发中能够注意这些细节，避免趟坑。

下一节将讲述浅拷贝和深拷贝的相关知识。



#### 课后题

给出一个 `PersonTransit` 类，一个 `Address` 类，假设 `Address` 是其它 jar 包中的类，没实现序列化接口。请使用今天讲述的自定义的函数 `writeObject` 和 `readObject` 函数实现 `PersonTransit` 对象的序列化，要求反序列化后 `address` 的值正常。

```java
@Data
public class PersonTransit implements Serializable {

    private Long id;
    private String name;
    private Boolean male;
    private List<PersonTransit> friends;
    private Address address;
}

@Data
@AllArgsConstructor
public class Address {
    private String detail;
  }
```

### 学习浅拷贝和深拷贝的正确方式

#### 前言

《手册》第 10 页有关于 `Object` 的 `clone` 问题的描述 [1](https://www.imooc.com/read/55/article/1141#fn1)：

> 【推荐】慎用 Object 的 clone 方法来拷贝对象。
> 说明：对象 clone 方法默认是浅拷贝，若想实现深拷贝需覆写 clone 方法实现域对象的深度遍历式拷贝。

那么我们要思考几个问题：

1. 什么是浅拷贝？
2. 浅拷贝和深拷贝的区别是什么？
3. 拷贝的目的是什么？
4. 拷贝的使用场景是什么？
5. 如何实现深拷贝？

网上也有很多介绍浅拷贝和深拷贝的文章，但文章质量参差不齐，有些文章读完仍然对概念得理解非常含糊。读完这些文章对拷贝的使用场景，对深拷贝的实现方式等都无法有全面和深刻的理解。

为此本节将带着大家系统地研究这上述问题，以便大家未来遇到类似问题时可以举一反三，灵活迁移。



#### 概念介绍



##### 拷贝 / 克隆的概念

我们先研究第 1 个问题：**什么是拷贝？**

维基百科对 “克隆” 的描述如下 [2](https://www.imooc.com/read/55/article/1141#fn2)：

> 克隆 (英语： Clone) 在广义上是指利用生物技术由无性生殖产生与原原个体有完全相同基因组之后代的过程。
>
> 在园艺学上，克隆指通过营养繁殖产生的单一植株的后代，很多植物都是通过克隆这样的无性繁殖方式从单一植株获得大量的子代个体。
>
> 在生物学上，是指选择性地复制出一段 DNA 序列（分子克隆）、细胞（细胞克隆）或个体（个体克隆）。
>
> 克隆一个生物体意味着创造一个与原先的生物体具有完全一样的遗传信息的新生物体。

计算机中的拷贝或克隆和上述概念很类似，可以类比理解。

对象的拷贝，就是根据原来的对象 “复制” 一份属性、状态一致的新的对象。



##### 为什么需要拷贝方法？

我们思考第 2 个问题：**为什么需要拷贝呢？**

我们来看下面的订单类和商品类。

订单类（ `Order` ）：

```java
@Data
public class Order {

    private Long id;

    private String orderNo;

    private List<Item> itemList;
}
```

商品类（ Item` ）：

```java
@Data
public class Item {
    private Long id;

    private Long itemId;

    private String name;

    private String desc;

    // 省略其他
}
```

如果我们查询得到 1 个订单对象，该对象包括 6 个商品对象。

如果我们还需要构造多个新的订单对象，属性和上述订单对象非常相似，只是订单号不同或者商品略有区别。

这时如果有一个 “复制” 方法，可以将订单复制一个副本，而且修改副本中的订单号和商品列表 ( `itemList` ) 不影响原始对象，是不是很方便？

另外一个非常典型的场景是在多线程中。如果只用一个主线程，在主线程中修改订单号分别调用 `doSomeThing` 函数，想分别打印 first 和 second 两个订单编号字符串。

```java
@Slf4j
public class CloneDemo { 
  
  public static void main(String[] args) {
        Order order = OrderMocker.mock();
        order.setOrderNo("first");
        doSomeThing(order);
        order.setOrderNo("second");
        doSomeThing(order);
    }

   private static void doSomeThing(Order order) {
        try {
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(order.getOrderNo());
   }
}
```

运行程序后输出的结果的确是: `first`、`second`。

但在多线程环境中，如果我们不通过克隆构造新的对象，线程池中两个线程会公用同一个对象，后面对订单号的修改将影响到其它线程。

```java
@Slf4j
public class CloneDemo {
  
	public static void main(String[] args) {
	    ExecutorService executorService = Executors.newFixedThreadPool(5);
	    Order order = OrderMocker.mock();
	    order.setOrderNo("first");
	    executorService.execute(() -> doSomeThing(order));
	    order.setOrderNo("second");
	    executorService.execute(() -> doSomeThing(order));

	}

	private static void doSomeThing(Order order) {
	    try {
	        TimeUnit.SECONDS.sleep(1L);
	    } catch (InterruptedException e) {
	        e.printStackTrace();
	    }
	    System.out.println(order.getOrderNo());
	}
}
```

输出的结果是: `second`、`second`。

因此如果能够克隆一个新的对象，并且对新对象的修改不影响原始对象，就能实现我们期待的效果。



##### 什么是浅拷贝？浅拷贝和深拷贝的区别是什么？

通过前言部分的介绍，我们知道 `Object` 的 `clone` 函数默认是浅拷贝。

按照惯例我们进入源码，看看是否能够得到我们想要的答案：

```java
/**
 * Creates and returns a copy of this object.  The precise meaning
 * of "copy" may depend on the class of the object. The general
 * intent is that, for any object {@code x}, the expression:
 * <blockquote>
 * <pre>
 * x.clone() != x</pre></blockquote>
 * will be true, and that the expression:
 * <blockquote>
 * <pre>
 * x.clone().getClass() == x.getClass()</pre></blockquote>
 * will be {@code true}, but these are not absolute requirements.
 * While it is typically the case that:
 * <blockquote>
 * <pre>
 * x.clone().equals(x)</pre></blockquote>
 * will be {@code true}, this is not an absolute requirement.
 * <p>
 * By convention, the returned object should be obtained by calling
 * {@code super.clone}.  If a class and all of its superclasses (except
 * {@code Object}) obey this convention, it will be the case that
 * {@code x.clone().getClass() == x.getClass()}.
 * <p>
 * By convention, the object returned by this method should be independent
 * of this object (which is being cloned).  To achieve this independence,
 * it may be necessary to modify one or more fields of the object returned
 * by {@code super.clone} before returning it.  Typically, this means
 * copying any mutable objects that comprise the internal "deep structure"
 * of the object being cloned and replacing the references to these
 * objects with references to the copies.  If a class contains only
 * primitive fields or references to immutable objects, then it is usually
 * the case that no fields in the object returned by {@code super.clone}
 * need to be modified.
 * <p>
 * The method {@code clone} for class {@code Object} performs a
 * specific cloning operation. First, if the class of this object does
 * not implement the interface {@code Cloneable}, then a
 * {@code CloneNotSupportedException} is thrown. Note that all arrays
 * are considered to implement the interface {@code Cloneable} and that
 * the return type of the {@code clone} method of an array type {@code T[]}
 * is {@code T[]} where T is any reference or primitive type.
 * Otherwise, this method creates a new instance of the class of this
 * object and initializes all its fields with exactly the contents of
 * the corresponding fields of this object, as if by assignment; the
 * contents of the fields are not themselves cloned. Thus, this method
 * performs a "shallow copy" of this object, not a "deep copy" operation.
 * <p>
 * The class {@code Object} does not itself implement the interface
 * {@code Cloneable}, so calling the {@code clone} method on an object
 * whose class is {@code Object} will result in throwing an
 * exception at run time.
 *
 * @return     a clone of this instance.
 * @throws  CloneNotSupportedException  if the object's class does not
 *               support the {@code Cloneable} interface. Subclasses
 *               that override the {@code clone} method can also
 *               throw this exception to indicate that an instance cannot
 *               be cloned.
 * @see java.lang.Cloneable
 */
protected native Object clone() throws CloneNotSupportedException;
```

该函数给出了非常详尽的介绍。下面给出一些要点的翻译：

> 该方法是创建对象的副本。这就意味着 “副本” 依赖于该对象的类型。
>
> 对于任何对象而言，一般来说下面的表达式成立：
>
> `x.clone() != x` 的结果为 `true` 。
>
> `x.clone().getClass() == x.getClass()` 的结果为 `true` 。
>
> 但是这些也不是强制的要求。
>
> `x.clone().equals(x)` 的结果也是 `true`。这也不是强制要求。
>
> 按照惯例，返回对象应该通过调用 `super.clone` 函数来构造。如果一个类和它的所有父类（除了 `Object` ）都遵循这个约定，那么 `x.clone().getClass() == x.getClass()` 将成立。
>
> 按照惯例，返回的对象应该和原始对象是独立的。
>
> 为了实现这种独立性，后续应该在调用 `super.clone` 得到拷贝对象并返回之前，应该对内部深层次的可变对象创建副本并指向克隆对象的对应属性的引用。
>
> 如果一个类只包含基本类型的属性或者指向不可变对象的引用，这种情况下，`super.clone` 返回的对象不需要被修改。
>
> 如果调用 `clone` 函数的类没有实现 `Cloneable` 接口将会抛出 `CloneNotSupportedException`。
>
> 注意所有的数组对象都默认实现了 `Cloneable` 接口。
>
> 该函数会创建该类的新实例，并初始化所有属性对象。属性对象本身并不会自动调用 `clone`。
>
> 因此此方法实现的是浅拷贝而不是深拷贝。

因此我们可以了解到，浅拷贝将返回该类的新的实例，该实例的引用类型对象共享。
深拷贝也会返回该类的新的实例，但是该实例的引用类型属性也是拷贝的新对象。

如果用一句话来描述，**浅拷贝和深拷贝的主要区别在于对于引用类型是否共享。**
![图片描述](img/5db65a6400019de012360594.png)
为了更好地理解浅拷贝，我们给出一个示例：

改造订单对象：

```java
@Data
public class Order implements Cloneable {

    private Long id;

    private String orderNo;

    private List<Item> itemList;

    @Override
    public Order clone() {
        try {
            return (Order)super.clone();
        } catch (CloneNotSupportedException ignore) {
            // 不会调到这里
        }
        return null;
    }
}
```

通过 `Object` 类的 `clone` 函数的注释我们了解到：如果调用 `clone` 函数的类没有实现 `Cloneable` 接口将会抛出 `CloneNotSupportedException` 。

因此要实现 `Cloneable` 接口。

重写 `clone` 函数是为了供外部使用，因此定义为 `public` 。

返回值类型定义为客户端直接需要的对象类型（本类）。

这体现了《Effective Java》的 Item 11 中所提到的 [3](https://www.imooc.com/read/55/article/1141#fn3)：

> Never make the client do anything the library can do for the client.
>
> 不要让客户端去做任何类库可以替它完成的事。

我们为上述浅拷贝编写测试代码：

```java
public class OrderMocker {

    public static Order mock() {
        Order order = new Order();
        order.setId(1L);
        order.setOrderNo("abcdefg");
        List<Item> items = new ArrayList<>();
        Item item = new Item();
        item.setId(0L);
        item.setItemId(0L);
        item.setName("《阿里巴巴Java开发手册》详解慕课专栏");
        item.setDesc("精品推荐");
        items.add(item);
        order.setItemList(items);
        return order;
    }
}
 @Test
  public void shallowClone() {
      Order order = OrderMocker.mock();
      Order cloneOrder = order.clone();

      assertFalse(order == cloneOrder);
      assertTrue(order.getItemList() == cloneOrder.getItemList());
  }
```

该单元测试可以通过，从而证实了 `clone` 函数的注释，证实了浅拷贝的表现。

即浅拷贝后，原对象的订单列表和克隆对象的订单列表地址相同。

**因此如果使用浅拷贝，修改拷贝订单的商品列表，那么原始订单对象的商品列表也会受到影响。**

为了更形象地理解浅拷贝和深拷贝的概念，我们以文件夹进行类比：

> 浅拷贝：同一个文件夹的两个快捷方式，虽然是两个不同的快捷方式，但是指向的文件夹是同一个，不管是通过哪个快捷方式进入，对该文件夹下的文件修改，相互影响。
>
> 深拷贝：我们复制某个文件夹（含里面的内容）在另外一个目录进行粘贴，就可得到具有相同内容的新目录，对新文件夹修改不影响原始文件夹。



#### 深拷贝的实现方式

虽然浅拷贝能够实现拷贝的功能，但是浅拷贝的引用类型成员变量是共享的，修改极可能导致相互影响。

业务开发中使用深拷贝更多一些，**那么实现深拷贝有哪些方式呢？**



##### 手动深拷贝

```java
@Data
public class Order implements Cloneable {

    private Long id;

    private String orderNo;

    private List<Item> itemList;


    @Override
    public Order clone() {
        try {
            Order order = (Order) super.clone();
            if (id != null) {
                order.id = new Long(id);
            }
            if (orderNo != null) {
                order.orderNo = new String(orderNo);
            }

            if (itemList != null) {
                List<Item> items = new ArrayList<>();
                for (Item each : itemList) {
                    Item item = new Item();
                    Long id = each.getId();
                    if(id != null){
                        item.setId(new Long(id));
                    }
                    Long itemId = each.getItemId();
                    if(itemId != null){
                        item.setItemId(new Long(itemId));
                    }
                    String name = each.getName();
                    if(name != null){
                        item.setName(new String(name));
                    }
                    String desc = each.getDesc();
                    if(desc != null){
                        item.setDesc(new String(desc));
                    }
                    items.add(item);
                }
                order.setItemList(items);
            }
            return order;
        } catch (CloneNotSupportedException ignore) {

        }

        return null;
    }
}
```

深拷贝也调用 `super.clone` 是为了支撑 `x.clone().getClass() == x.getClass()` 。

写好代码后，通过调用 `Order` 类的 `clone` 函数即可实现深拷贝。

由于克隆的对象和内部的引用类型的属性全部都是依据原始对象新建的对象，因此如果修改拷贝对象的商品列表，原始订单对象的商品列表并不会受到影响。

通过下面的单元测试来验证：

```java
@Test
public void deepClone() {
    Order order = OrderMocker.mock();
    Order cloneOrder = (Order) order.clone();

    assertFalse(order == cloneOrder);
    assertFalse(order.getItemList() == cloneOrder.getItemList());
}
```

该单测可顺利通过。



##### 序列化方式

前面章节我们讲到了序列化和反序列化的知识，讲到了序列化的主要使用场景包括深拷贝。

序列化通过将原始对象转化为字节流，再从字节流重建新的 Java 对象，因此原始对象和反序列化后的对象修改互不影响。

因此可以使用之前讲到的序列化和反序列化方式来实现深拷贝。



###### 自定义序列化工具函数

如果我们不想为了深拷贝这一项功能就依赖新的 jar 包，可以在自己项目中借助对象输入和输出流编写拷贝工具函数。

示例代码如下：

```java
  /**
     * JDK序列化方式深拷贝
     */
    public static <T> T deepClone(T origin) throws IOException, ClassNotFoundException {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        try (ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);)        {
            objectOutputStream.writeObject(origin);
            objectOutputStream.flush();
        }
        byte[] bytes = outputStream.toByteArray();
        try (ByteArrayInputStream inputStream = new ByteArrayInputStream(bytes);) {
            return JdkSerialUtil.readObject(inputStream);
        }
    }
```

我们可通过调试查看克隆对象和原始对象。

从下图中我们可以清晰地看到，通过此方法克隆得到的新的对象是一个全新的对象。

![图片描述](img/5db65a8800015e6210170716.png)
需要注意的是：正如前面章节所讲，Java 序列化需要实现 `Serializable` 接口，而且效率不是特别高。



###### commons-lang3 的序列化工具类

我们可以利用项目中引用的常见工具包的工具类实现深拷贝，避免重复造轮子。

可以使用 commons-lang3 （3.7 版本）的序列化工具类： `org.apache.commons.lang3.SerializationUtils#clone`。

用法非常简单：

```java
@Test
public void serialUtil() {
    Order order = OrderMocker.mock();
   // 使用方式
    Order cloneOrder = SerializationUtils.clone(order);

    assertFalse(order == cloneOrder);
    assertFalse(order.getItemList() == cloneOrder.getItemList());
}
```

前面反复提到过，我们学习知识不仅要知其然，而且要知其所以然。

**那么它是如何实现深拷贝的呢？**

按照惯例我们打开源码：

```java
/**
 * <p>Deep clone an {@code Object} using serialization.</p>
 *
 * <p>This is many times slower than writing clone methods by hand
 * on all objects in your object graph. However, for complex object
 * graphs, or for those that don't support deep cloning this can
 * be a simple alternative implementation. Of course all the objects
 * must be {@code Serializable}.</p>
 *
 * @param <T> the type of the object involved
 * @param object  the {@code Serializable} object to clone
 * @return the cloned object
 * @throws SerializationException (runtime) if the serialization fails
 */
public static <T extends Serializable> T clone(final T object) {
    if (object == null) {
        return null;
    }
    final byte[] objectData = serialize(object);
    final ByteArrayInputStream bais = new ByteArrayInputStream(objectData);

    try (ClassLoaderAwareObjectInputStream in = new ClassLoaderAwareObjectInputStream(bais,
            object.getClass().getClassLoader())) {
        /*
         * when we serialize and deserialize an object,
         * it is reasonable to assume the deserialized object
         * is of the same type as the original serialized object
         */
        @SuppressWarnings("unchecked") // see above
        final T readObject = (T) in.readObject();
        return readObject;

    } catch (final ClassNotFoundException ex) {
        throw new SerializationException("ClassNotFoundException while reading cloned object data", ex);
    } catch (final IOException ex) {
        throw new SerializationException("IOException while reading or closing cloned object data", ex);
    }
}
```

通过其返回值的泛型描述 `` 可以断定参数对象需要实现序列化接口。

该函数注释也给出了性能说明，该深拷贝方法性能不如直接手动写 `clone` 方法效率高。

大家可以进到该方法的子函数中查看更多细节。

通过源码的分析我们发现，该克隆函数本质上也是通过 Java 序列化和反序列化方式实现。



###### JSON 序列化

我们还可以通过 JSON 序列化方式实现深拷贝。

下面我们利用 Google 的 Gson 库（2.8.5 版本），实现基于 JSON 的深拷贝：

首先我们将深拷贝方法封装到拷贝工具类中：

```java
/**
 * Gson方式实现深拷贝
 */
public static <T> T deepCloneByGson(T origin, Class<T> clazz) {
    Gson gson = new Gson();
    return gson.fromJson(gson.toJson(origin), clazz);
}
```

使用时直接调用封装的工具方法即可：

```java
 @Test
 public void withGson() {
      Order order = OrderMocker.mock();
      // gson序列化方式
      Order cloneOrder = CloneUtil.deepCloneByGson(order, Order.class);

      assertFalse(order == cloneOrder);
      assertFalse(order.getItemList() == cloneOrder.getItemList());
  }
```

使用 JSON 序列化方式实现深拷贝的好处是，性能比 Java 序列化方式更好，更重要的是不要求序列化对象以及成员属性（嵌套）都要实现序列化接口。

我们也可以使用前面讲到的 Hessian 和 Kryo 序列化来实现，请大家自行封装。

上面通过 Gson 实现的深拷贝工具方法封装，再次体现了 “不要让客户端去做任何类库可以替它完成的事” 的原则。

这点也和《重构 - 改善既有代码的设计》 第一版 10.13 封装向下转型的重构方案一致。

最后，建议**不管采取哪种或者哪几种深拷贝方式，都尽量将其封装到项目的克隆工具类中，方便复用**。



#### 总结

本节重点讲述了浅拷贝和深拷贝的概念，它们的主要区别，以及浅拷贝和深拷贝的实现方式。

下一节将讲述开发常用既熟悉又陌生的几种分层领域模型，讲述它们之间的区别和实际开发中的使用。



#### 课后题

请自定义一个类，编写代码分别实现浅拷贝和深拷贝。

### 分层领域模型使用解读

#### 前言

《手册》关于分层模型部分的规约如下 [1](https://www.imooc.com/read/55/article/1142#fn1)：

> 【参考】分层领域模型规约
> DO (Data Object): 此对象与数据库表结构一一对应，通过 DAO 层向上传输数据源对象。
>
> DTO (Data Transfer Object): 数据传输对象，Service 或 Manager 向外传输的对象。
>
> BO (Business Object): 业务对象，由 Service 层输出的封装业务逻辑的对象。
>
> AO (Application Object): 应用对象，在 Web 层与 Service 层之间抽象的复用对象模型，极为贴 近展示层，复用度不高。
>
> VO (View Object): 显示层对象，通常是 Web 向模板渲染引擎层传输的对象。Query: 数据查询对象，各层接收上层的查询请求。
>
> 注意超过 2 个参数的查询封装，禁止使用 Map 类来传输。

那么我们需要思考以下几个问题：

- 为什么需要这些分层领域模型？
- 实际开发中每种分层领域模型都会用到吗？

本小节我们将重点分析和解答这些问题。



#### 分层模型



##### 常见的分层模型有哪些？含义是什么？

学习和工作经常会接触到分层领域模型，如 DO、BO、DTO、VO 等。其中 DO、BO、DTO、AO、Query 在《手册》给出了一些解释，这里给出一些补充。

DTO (Data Transger Object) 为数据传输对象，通常将底层的数据聚合传给外部系统，它通常用作 Service 和 Manager 层向上层返回的对象。需要注意的是：如果作为分布式服务的参数或返回对象，通常要实现序列化接口。

Param 为查询参数对象，适用于各层，通常用作接受前端参数对象。Param 和 Query 的出现是为了避免使用 `Map` 作为接收参数的对象。

BO (Bussiness Object) 即业务对象。该对象中通常包含业务逻辑。此对象在实际使用中有不同的理解，有的团队采用领域驱动设计，BO 含有属性和方法（具体可参考领域驱动设计的相关图书）；有的团队将 BO 当做 Service 返回给上层的 “专用 DTO” 使用；而有的团队则当做 Service 层内保存中间信息数据的 “DTO” 或者上下文对象来使用（本文采用这种理解）。

比如 BO 中可以保存中间状态，放一些逻辑等，这些并不适合放在 `DTO` 中：

```java
@Data
public class ItemBO {

    private Boolean isOnSell;

    private Boolean hasStock;

    private Boolean hasSensitiveWords;

    public Boolean isLegal() {
        if (isOnSell == null || hasStock == null || hasSensitiveWords == null) {
            return false;
        }
        return isOnSell && hasStock && (!hasSensitiveWords);
    }
}
```

VO (View Object) 为视图对象，通常作为控制层通过 JSON 返回给前端然后前端渲染或者加载页面模板在后端进行填充。

AO (Application Object) 应用对象。通常用在控制层和服务层之间。有些团队会将前端查询的属性和保存的属性几乎一致的对象封装为 AO，如读取用户属性传给前端，用户在前端编辑了用户属性后传回后端。这种用法将 AO 用作 Param 和 VO 或 Param 和 DTO 的组合。



##### 为什么要有分层领域模型？

还有的朋友查询参数喜欢通过 `Map` 或者 `JSONObject` 来封装。有些朋友可能会认为这么多模型没有必要，因为通常各层模型的属性基本相同，而且各种类型的分层模型对象转换非常麻烦。

使用不同的分层领域模型能够让程序更加健壮、更容易拓展，可以降低系统各层的耦合度。

分层模型的优势只有在系统较大时才体现得更加明显。设想一下如果我们不想定义 DTO 和 VO，直接将 DO 用到数据访问层、服务层、控制层和外部访问接口上。此时该表删除或则修改一个字段，DO 必须同步修改，这种修改将会影响到各层，这并不符合高内聚低耦合的原则。通过定义不同的 DTO 可以控制对不同系统暴露不同的属性，通过属性映射还可以实现具体的字段名称的隐藏。不同业务使用不同的模型，当一个业务发生变更需要修改字段时，不需要考虑对其它业务的影响，如果使用同一个对象则可能因为 “不敢乱改” 而产生很多不优雅的兼容性行为。

如果我们不愿意定义 Param 对象，使用 Map 来接收前端的参数，获取时如果采用 JSON 反序列化，则可能出现上一节所讲到的反序列化类型丢失问题。如果我们不使用 Query 对象而是 `Map` 对象来封装 DAO 的参数，设置和获取的 `key` 很可能因为粗心导致设置和获取时的 key 不一致而出现 BUG。



#### 开发中的应用

讲完了概念和优势，大家可能会认为文字描述有些抽象，接下来通过查询和返回两个视角为大家展示实际项目中的一种常见的用法（贫血模型）。



##### 查询视图

我们先从请求访问的视角去了解不同分层数据模型在实际项目中一种常见用法。

![图片描述](img/5db65bbc0001a1da18681000.png)
前端或者其它服务将 `Param` 对象作为参数传给控制层或者对外服务接口，然后调用内部的服务类，服务类内部的中间数据和这些数据相关的逻辑可以封装为 `BO` ，比如根据 `BO` 多个属性判断是否符合某个条件。

如果查询数据则封装为 `Query` 对象作为参数，如果需要查询其它依赖，则可以封装 `Param` 对象作为参数去查询。`DAO` 层一般插入和更新的参数对象使用 `DO` 或 `Param`, 查询参数一般使用 `Query`，删除参数一般使用 `Param`。



##### 返回视图

接下来我们从数据返回的视角去了解分层领域模型在实际项目中的一种常见用法：

![图片描述](img/5db65bcd0001942c17701038.png)
数据访问层通常将数据封装为 `DO` 对象传给 `Service` 层，`Manager` 或 `Client` 层往往将查询结果封装为 `DTO` 传给 `Service` 层。

通常内部服务层通过 `DTO` 往外传输数据。`Controller` 通常将 `DTO` 组装为前端需要的 `VO` 或者直接将 `DTO` 外传 。

RPC 服务接口将 `DTO` 直接返回或者重新封装为新的 `DTO` 返回给外部服务。

另外即使同一个接口，但是一个对内使用，一个对外暴露，尽量使用不同接口，定义不同的参数和返回值，从而避免因为修改内部或外部的数据结构而导致另外一个受到影响，这也是单一职责原则的要求。

> **单一职责原则**：一个类应该有且只有一个改变的理由。

也有部分团队 RPC 的请求和响应参数都通过 DTO 来承载，通过 `XXRequestDTO` 和 `XXResponseDTO` 来表示。

实践分层领域模型能够提高项目的健壮性、可拓展性和可维护性，降低了系统内部各层的耦合度。

上面只是给出一种参考，很多团队对部分分层模型的理解会有差异，实际的使用过程中根据自己团队的规模可以适当变通。比如有很多团队项目并不是特别大，为了降低复杂度，只用到了 `DTO` 、`VO` 、`DO` 三种分层领域模型。

最后对分层领域模型的规约这里进行补充：

**【参考】不提倡在 DTO 中写逻辑，强制不要在 RPC 返回对象的 DTO 中封装逻辑。**

有些团队的个别成员会将根据成员属性作判断的一些函数写到 DTO 中，最奇葩的是该逻辑还主要供内部系统业务层使用。

如:

```java
public class xxDTO{

// 各种属性


// 逻辑代码
 public static boolean  canXXX(){ 
   // 各种判断
 }

}
```

这样造成系统的耦合性非常强。

如果对方用到了这个函数，未来此函数的内部逻辑必须发生变化，未必能及时通知对方升级，容易造成 BUG。

即使耗费了成本找到了使用方，为了你的功能，让别人被迫升级版本重新上线也是非常不专业的事情。

显然这样做不合理。

> 试想一下今天 A 部门告诉你他们因某个功能被迫修改了某个 RPC 返回值 DTO 的某个方法，你们用到没有？用到升级一下哈…
>
> 然后 B 部门的人明天告诉你同样的话，然后 C 部门，然后…
>
> 你会不会崩溃？

建议如果需要在内部业务中写对实体相关的逻辑，可以考虑封装到工具类 / 帮助类中。



#### 总结

本节主要讲分层模型的目的和优势以及在实际开发中的常见用法。给大家一个参考，让大家能够在开发时知道哪些模型应该放到哪一层。

下一节将讲述不同的分层领域模型之间的转换的正确姿势。



#### 思考题

在实际项目开发中，不同的分层领域模型之间通常需要转换，你是如何转换的？



### Java属性映射的正确姿势

#### 前言

前一节讲到项目为了更容易维护，易于拓展等原因会使用各种分层领域模型。在多层应用中，常需要对各种不同的分层对象进行转换，这就会存在一个非常棘手的问题即：编写不同的模型之间相互转换的代码非常麻烦。其中最常见和最简单的方式是编写对象属性转换函数，即普通的 Getter/Setter 方法。除此之外各种各种属性映射工具。

- 那么常见的 Java 属性映射工具有哪些？
- 它们的原理以及对其性能怎样？
- 实际开发中该如何选择？

本节将给出解答。



#### 常见的 Java 属性映射的工具及其原理



##### 常见的 Java 属性映射工具

常见的 Java 属性映射工具有以下几种：

1. `org.apache.commons.beanutils.BeanUtils#copyProperties`
2. `org.springframework.beans.BeanUtils#copyProperties(java.lang.Object, java.lang.Object)`
3. `org.dozer.Mapper#map(java.lang.Object, java.lang.Class)`
4. `net.sf.cglib.beans.BeanCopier#copy`
5. `ma.glasnost.orika.MapperFacade#map(S, D)`
6. `mapstruct`



##### 原理

1、Getter/Setter 方式使用原生的语法，虽然简单但是手动编写非常耗时；

2、通过 [dozer 的 maven 依赖](https://mvnrepository.com/artifact/net.sf.dozer/dozer/5.5.1)可以看出，dozer 并没有使用字节码增强技术，因为并没有引用任何字节码增强技术的 jar 包；

我们再从其核心类 `org.dozer.MappingProcessor` 中寻找线索：

```java
import java.lang.reflect.Array;
import java.lang.reflect.Modifier;
import java.lang.reflect.InvocationTargetException;
...
```

我们可以断定，dozer 使用的是反射机制。

3、同样的 commons 和 Spring 的 `BeanUtil` 工具类也采用的是反射方式。优点是两个是非常常用的类库，不需要引用更多复杂的包；

4、cglib 的 `BeanCopier` 的原理是不是也是反射机制呢？

我们可以通过 [cglib 的 maven 库](https://mvnrepository.com/artifact/cglib/cglib/3.2.12)的编译依赖中找到线索：

![图片描述](img/5db65c1900018c7d18440388.png)
发现该库依赖了 asm ，我们去 [asm 官网](https://asm.ow2.io/)可以看到它的介绍：

> asm 库是一个 Java 字节码操作和分析框架，它可以用来修改已经存在的字节码或者直接二进制形式动态生成 class 文件。asm 的特点是小且快。

、同样的，我们可以通过 [orika 的 maven 库](https://mvnrepository.com/artifact/ma.glasnost.orika/orika-core/1.5.4)得到其实现依赖的核心技术：

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)
其中 javassist 我们知道它是一个字节码操作工具。

我们去[它的](https://asm.ow2.io/)[官网](http://www.javassist.org/)看下介绍：

> javassist 让操作字节码非常容易。javassist 允许 java 程序运行时定义一个新的类，也可以实现在 JVM 加载类文件时修改它。javassist 提供两种级别的 API ，一种是源码级别；一种是字节码级别。使用源码级别的 API，无需对 java 字节码特定知识有深入的了解就可以轻松修改类文件。字节码级别的 API 则允许用户直接修改类文件。

6、通过 [MapStruct 的官网](https://mapstruct.org/)的介绍我们可以看出，mapstruct 采用原生的方法调用，因此更快速，更安全也更容易理解。根据官网的介绍我们知道，使用时只需要使用它的注解，定义好转换接口，转换函数，编译时会自动生成转换工具的实现类、调用属性赋值和取值函数实现转换。mapstruct 还支持通过注解形式定义不同属性名的映射关系等，功能很强大。

转换代码：

```java
@Mapper
public interface UserMapper {
    UserMapper INSTANCE = (UserMapper)Mappers.getMapper(UserMapper.class);

    UserDTO userDo2Dto(UserDO var1);
}
```

编译后生成自动的转换接口的实现类：

```java
public class UserMapperImpl implements UserMapper {
    public UserMapperImpl() {
    }

    public UserDTO userDo2Dto(UserDO userDO) {
        if (userDO == null) {
            return null;
        } else {
            UserDTO userDTO = new UserDTO();
            userDTO.setName(userDO.getName());
            userDTO.setAge(userDO.getAge());
            userDTO.setNickName(userDO.getNickName());
            userDTO.setBirthDay(userDO.getBirthDay());
            return userDTO;
        }
    }
}
```

大大简化了代码。

官方还提供了非常详细的[参考文档](https://mapstruct.org/documentation/reference-guide/) 和使用范例，提供了很多高级用法。



##### 性能

接下来按照惯例，我们对比一下它们的性能。

我们在 `com.imooc.basic.converter.UserConverterTest` 类中对上面的常见对象转换方式进行单测 `UserDO` 对象：

```java
@Data
public class UserDO {
    private Long id;
    private String name;
    private Integer age;
    private String nickName;
    private Date birthDay;
}
```

目标对象：

```java
@Data
public class UserDTO {
    private String name;
    private Integer age;
    private String nickName;
    private Date birthDay;
}
```

使用 easyrandom（后面的单元测试环节会重点介绍）构造 10 万个 `UserDO` 随机对象进行性能对比。spring 版本为 5.1.8.RELEASE，dozer 版本为 5.5.1，orika-core 版本为 1.5.4，cglib 版本为 3.2.12，commons-lang3 包版本为 3.9，10 次运行取平均值，最终结果如下：

1. 普通 Getter/Setter 耗时 365ms；
2. `org.apache.commons.beanutils.BeanUtils#copyPropertie` 耗时 9s273ms；
3. `org.springframework.beans.BeanUtils#copyProperties(java.lang.Object, java.lang.Object)` 耗时 2s327ms；
4. `org.dozer.Mapper#map(java.lang.Object, java.lang.Class)` 耗时 9s271ms；
5. `ma.glasnost.orika.MapperFacade#map(S, D)` 耗时 837ms；
6. `net.sf.cglib.beans.BeanCopier#copy` 耗时 409ms；
7. MapStruct 393ms。

![图片描述](img/5db65c4100013ce912961116.png)
由于机器的性能不同结果会有偏差，本实验并没有将转换框架的功能发挥到到极致，也没有使用更复杂的对象进行对比，因此本实验的结果仅作为一个大致的参考。

我们仍然可以大致可以得出结论：采用字节码增强技术的 Java 属性转换工具和普通的 Getter/Setter 方法性能相差无几，甚至比 Getter/Setter 效率还高，反射的性能相对较差。

因此从性能来讲首推 Getter/Setter 方式（含 MapStruct），其次是 cglib。



#### 用哪个？为什么？怎么用？



##### 用什么？为什么？

通过以上的分析，我们对 Java 属性转换有了一个基本的了解。

**选择太多往往会比较纠结，实际开发中我们用哪种更好呢？**

我在业务代码中见到同事用的转换工具主要有 Getter/Setter 方式、 orika 和 commons/spring 的属性拷贝工具。

**属性转换工具的优势**：用起来方便，往往一行行代码就实现多属性的转换，而且属性不对应可以通过注解或者修改配置方式自动适配，功能非常强大。

**属性转换工具的缺点**：

1. 多次对象映射（从 A 映射到 B，再从 B 映射到 C）如果属性不完全一致容易出错；
2. 有些转换工具，属性类型不一致自动转换容易出现意想不到的 BUG；
3. 基于反射和字节码增强技术的映射工具实现的映射，对一个类属性的修改不容易感知到对其它转换类的影响。

**我们可以想想这样一个场景**：

> 一个 `UserDO` 如果属性超多，转换到 `UserDTO` 再被转换成 `UserVO` 。如果你修改 `UserDTO` 的一个属性命名，其它类待映射的类新增的对应属性有一个字母写错了，编译期间不容易发现问题，造成 BUG。
>
> 如果使用原始的 Getter/Setter 方式转换，修改了 `UserDO` 的属性，那么转换代码就会报错，编译都不通过，这样就可以逆向提醒我们注意到属性的变动的影响。

**因此强烈建议使用定义转换类和转换函数，使用插件实现转换，不需要引入其它库，降低了复杂性，可以支持更灵活的映射。**

大家可以想想这种场景：

> 如果一个 A 映射到 B，B 有两个属性来自 C，一个属性来自于传参或者计算等。

此时自定义转换函数就更方便。

**如果使用属性映射工具推荐使用 MapStruct，更安全一些，转换效率也很高。**



##### 怎么用？

每种对象属性映射工具的具体用法，大家可以参考官网文档或源码中的测试类，这里主要讲映射的工具类该如何定义。

为了避免转换函数散落到多个业务类中，不容易复用，我们可以在工具包或者对象包下定义一个专门的转换包（converter 或者 mapper 包），在转换的包下编写转换工具类。

**第一种方式**：可以实现 `org.springframework.core.convert.converter.Converter` 接口。

代码如下：

```java
import org.springframework.core.convert.converter.Converter;

public class UserDO2DTOConverter implements Converter<UserDO, UserDTO> {

    @Override
    public UserDTO convert(UserDO source) {
        UserDTO userDTO = new UserDTO();
        userDTO.setName(source.getName());
        userDTO.setAge(source.getAge());
        userDTO.setNickName(source.getNickName());
        userDTO.setBirthDay(source.getBirthDay());
        return userDTO;
    }
}
```

上述只能实现单向转换，**我们如果想双向转换该怎么做呢？**

这时候我们可以采用**第二种方式**，可以继承 `com.google.common.base.Converter` 接口实现双向转换。

```java
import com.imooc.basic.converter.entity.UserDO;
import com.imooc.basic.converter.entity.UserDTO;
import com.google.common.base.Converter;

public class UserDO2DTOConverter extends Converter<UserDO, UserDTO> {

    @Override
    protected UserDTO doForward(UserDO userDO) {
        UserDTO userDTO = new UserDTO();
        userDTO.setName(userDO.getName());
        userDTO.setAge(userDO.getAge());
        userDTO.setNickName(userDO.getNickName());
        userDTO.setBirthDay(userDO.getBirthDay());
        return userDTO;

    }

    @Override
    protected UserDO doBackward(UserDTO userDTO) {
        UserDO userDO = new UserDO();
        userDO.setName(userDTO.getName());
        userDO.setAge(userDTO.getAge());
        userDO.setNickName(userDTO.getNickName());
        userDO.setBirthDay(userDTO.getBirthDay());
        return userDO;

    }
  }
```

我更建议采用以下这种方式，因为上述方式只能实现单向或者双向转换，如果更多种对象类型的转换就无能为力。

此时可以自定义接口或者抽象类，支持更多种对象的转换。

更推荐大家直接定义某个对象的转换器类，在其内部编写该对象各层对象的转换函数：

```java
public class UserConverter {

    public static UserDTO convertToDTO(UserDO source) {
        UserDTO userDTO = new UserDTO();
        userDTO.setName(source.getName());
        userDTO.setAge(source.getAge());
        userDTO.setNickName(source.getNickName());
        userDTO.setBirthDay(source.getBirthDay());
        return userDTO;
    }

    public static UserDO convertToDO(UserDO source) {
        UserDO userDO = new UserDO();
        userDO.setId(source.getId());
        userDO.setName(source.getName());
        userDO.setAge(source.getAge());
        userDO.setNickName(source.getNickName());
        userDO.setBirthDay(source.getBirthDay());
        return userDO;
    }
    
  // 转换成UserVO等
}
```

有些同学可能会抱怨，**Getter/Setter 方式转换函数编写非常耗时而且容易漏，怎么办？**

这里推荐一个 IDEA 插件：**GenerateAllSetter** 或者 **GenerateO2O**。

定义好转换函数之后，鼠标放在 `convertToDTO` 上使用快捷键，选择 “generate setter getter converter” 即可实现根据目标对象的属性名适配同名源对象自动填充，注意如果有个别属性不对应，需手动转换。

**另外推荐使用 mapstruct 实现对象属性映射**：

```java
@Mapper
public interface UserMapper {
    UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

    UserDTO  userDo2Dto(UserDO userDO);
}
```

使用时一行代码即可搞定：

```java
UserDTO userDTO = UserMapper.INSTANCE.userDo2Dto(userDO);
```

相当于把 IDE 插件自动生成的这部分任务改为了使用注解，通过插件编译时自动生成。



#### 总结

本节主要介绍了 Java 属性映射的各种方式，介绍了每种方式背后的原理，并简单对比了各种属性映射方式的耗时。本小节还给出了属性转换工具的推荐定义方式。希望大家在实际的开发中，除了考虑性能外，兼顾考虑安全性和可维护性。

下节将介绍过期代码的正确处理方式。



#### 课后练习

自定义一个 OrderDO 和 OrderDTO 两个类，自定义属性，使用 StructMap 实现属性映射。



### 过期类、属性、接口的正确处理姿势



#### 前言

《手册》第 7 页对于过时类有这样一句描述 [1](https://www.imooc.com/read/55/article/1144#fn1)：

> 接口过时必须加 @Deprecated 注解，并清晰地说明采用的新接口或者新服务是什么。
> 接口提供方既然明确是过时接口，那么有义务同时提供新的接口；作为调用方来说，有义务去考证过时方法的新实现是什么。

那么我们要思考为什么要这么做呢？这个指导原则如何更好地落地呢？



#### 为什么要这么做

如果有机会进入一个大一点的公司，而且你是一个有追求的人，你可能会遇到下面几种情况。

- 当你接手一个服务，看到某个类、属性、函数被标注为 @Deprecated 但是没有注释的时候，内心是崩溃的；
- 当你对接二方服务，升级 jar 包后发现使用的接口被标记为废弃但是没注释时，内心也是崩溃的；
- 当你看到同事封装的一些工具类使用了一些被废弃的类时，你的内心同样同样是崩溃的。不改放在那看着难受，改又无故得耗费自己的时间，而且还怕改出 BUG。

试想一下，如果你接手一个服务里面的类、属性和函数要被废弃了连 @Deprecated 都不加，是不是很容易 “放心” 使用进而被坑？

如果被标注为 `@Deprecated` ，给出注释说明为什么被废弃，新的接口是什么，心里会不会更踏实？

如果对接的二方服务 jar 包升级以后发现，使用的接口被废弃且给出详细的告诉你改用哪个新接口，是不是心里更有底？

试想一下如果我们每个人都能遵守这种规约，封装工具类时遇到过时的类，主动去学习并使用新的替换类，是不是就不会好很多？



#### 如何落实

那么，说了这么多，究竟该如何落地呢？
我认为：最好的学习方式之一就是找一些优秀的源码相关的示例进行学习。



##### JDK 的类或常见三方库

我们以 JDK 中的 `URLEncoder` 和 `URLDecoder` 为例介绍如何写过期函数的注释和如何替换该过期函数：

```java
String url = "xxx";
String encode = URLEncoder.encode(url);
log.debug("URL编码结果：" + encode);
String decode = URLDecoder.decode(encode);
log.debug("URL解码结果：" + decode);
```

在 IDEA 中编写如上代码时候，`java.net.URLEncoder#encode(java.lang.String)` 和 `java.net.URLDecoder#decode(java.lang.String)` 会有删除的标志，便表示该函数已经过期。

那么如何找到新函数和修改呢？

我们进到源码里查看:

```java
/**
 * Decodes a {@code x-www-form-urlencoded} string.
 * The platform's default encoding is used to determine what characters
 * are represented by any consecutive sequences of the form
 * "<i>{@code %xy}</i>".
 * @param s the {@code String} to decode
 * @deprecated The resulting string may vary depending on the platform's
 *          default encoding. Instead, use the decode(String,String) method
 *          to specify the encoding.
 * @return the newly decoded {@code String}
 */
@Deprecated
public static String decode(String s) {
    String str = null;
    try {
        str = decode(s, dfltEncName);
    } catch (UnsupportedEncodingException e) {
        // The system should always have the platform default
    }
    return str;
}
```

在 `@deprecated` 的注释里我们找到了答案：“The resulting string may vary depending on the platform’s default encoding.（解析结果的字符串和系统的默认字符编码强关联）”，并给出了替代函数的说明 “Instead, use the `decode(String,String)` method to specify the encoding.（使用 `decode(String,String)` 函数来指定字符串编码）”

因此我们提供新的接口，就得接口要废弃时也可以参考这里**写上废弃的原因以及替代的新接口**。

我们还可以通过 [codota](https://www.codota.com/code/query) 来搜索（建议在 IDEA 安装插件，使用更方便）看常见类库的常见函数的用法，甚至可以看到某些函数的使用概率：

![图片描述](img/5db65c9600011e4909680261.png)
搜索我们想要的类和方法：[URLEncoder.encode](https://www.codota.com/code/query/java.net@URLEncoder@encode)，即可得到 github 优秀的开源框架或 stackoverflow 中相关优秀范例。根据相关的优秀代码范例进行修改。

![图片描述](img/5db65c820001b48909150715.png)

我们改用新的函数：

```java
    String url = "xxx";
    String encode = URLEncoder.encode(url, Charsets.UTF_8.name());
    log.debug("URL编码结果：" + encode);
    String decode = URLDecoder.decode(encode, Charsets.UTF_8.name());
    log.debug("URL解码结果：" + decode);
```

对类似废弃的接口的改动，最好要使用单元测试进行验证：

```java
/**
 * 新旧两种接口对比
 *
 * @throws UnsupportedEncodingException
 */
@Test
public void testURLUtil() throws UnsupportedEncodingException {

    String url = "http://www.imooc.com/test?name=张三";
    // 旧的函数
    String encodeOrigin = URLEncoder.encode(url);
    String decodeOrigin = URLDecoder.decode(encodeOrigin);

    // 新的函数
    String encodeNew = URLEncoder.encode(url, Charsets.UTF_8.name());
    String decodeNew = URLDecoder.decode(encodeNew, Charsets.UTF_8.name());

    // 结果对比
    Assert.assertEquals(encodeOrigin, encodeNew);
    Assert.assertEquals(decodeOrigin, decodeNew);
}
```

如果是常见的**三方库**，也可以采用类似的步骤，一般都很快解决问题。

如我们发现下面的函数被废弃，进入到源码中查看：

```
org.springframework.util.Assert#doesNotContain(java.lang.String, java.lang.String)
/**
 * @deprecated as of 4.3.7, in favor of {@link #doesNotContain(String, String, String)}
 */
@Deprecated
public static void doesNotContain(String textToSearch, String substring) {
   doesNotContain(textToSearch, substring,
         "[Assertion failed] - this String argument must not contain the substring [" + substring + "]");
}
```

直接通过点击 `{@link #doesNotContain(String, String, String)` 可以快速进入新的替代函数去查看。

从这里例子我们还学到了一个新的技巧，如果是二方库或者三方库，废弃的属性、函数在注释中除了可以写原因和替代函数外，可以标注从哪个版本被标注为废弃。替代函数可以使用 `{@link}` 方式，更便捷和优雅。

再回顾上面 `java.net.URLDecoder#decode(java.lang.String)` 的注释就没有提供这种方式，跳转就不够方便。

另外大家还可以学习一下 `@see` 的用法，以及 `@see` 和 `{@link}` 的区别，后面专栏也会对注释做专门的讲解。

我们从这个例子还可以看到注释中并没有说明废弃的原因，作为读者你会发现有些摸不着头脑，心里嘀咕 “为啥被废弃？”。

通过替换函数以及注释我们可以猜测废弃的原因是：” 默认的提示文本不够优雅 “且即使断言通过，第三个参数字符串拼接仍然会执行，造成不必要字符串连接操作。这点有点类似于日志中不建议使用字符串拼接当做日志内容（可以采用占位符的方式）。

新的替换函数的注释除了给出功能介绍外，也给出了使用的范例：

```java
Assert.doesNotContain(name, "rod", "Name must not contain 'rod'");
```

这里给我们带来的启发是，**写工具类时如果能再注释上添加一些范例和结果，则会极大方便使用者**。

这点在 `commons-lang3` 和 `guava` 等开源工具库中随处可见，值得我们学习。

随手选取一个例子，大家感受一下：

```java
/**
 * <p>Strips whitespace from the start and end of a String  returning
 * {@code null} if the String is empty ("") after the strip.</p>
 *
 * <p>This is similar to {@link #trimToNull(String)} but removes whitespace.
 * Whitespace is defined by {@link Character#isWhitespace(char)}.</p>
 *
 * <pre>
 * StringUtils.stripToNull(null)     = null
 * StringUtils.stripToNull("")       = null
 * StringUtils.stripToNull("   ")    = null
 * StringUtils.stripToNull("abc")    = "abc"
 * StringUtils.stripToNull("  abc")  = "abc"
 * StringUtils.stripToNull("abc  ")  = "abc"
 * StringUtils.stripToNull(" abc ")  = "abc"
 * StringUtils.stripToNull(" ab c ") = "ab c"
 * </pre>
 *
 * @param str  the String to be stripped, may be null
 * @return the stripped String,
 *  {@code null} if whitespace, empty or null String input
 * @since 2.0
 */
public static String stripToNull(String str) {
    if (str == null) {
        return null;
    }
    str = strip(str, null);
    return str.isEmpty() ? null : str;
}
```

对于常见的三方库，还有一个不错的技巧：我们可以从 github 上拉取其源代码，然后找到某个类对应的单元测试类中，在单元测试模块可以找到对应的参考用法。还可以在源码中打断点，进行深入研究。希望大家可以亲自实践，会有更加深刻的体会。



##### 二方库

作为接口的使用者，如果使用二方库，发现使用的功能被标注为废弃。

如果是 maven 项目可以通过 maven 命令拉取其源码和 javadoc。

```
mvn dependency:sources -DdownloadSources=true -DdownloadJavadocs=true
```

如果是 gradle 项目，也可以使用插件下载源码，查看其将被废弃的原因。

如果没有标注原因并给出替代方案，或给出的注释不够详细，建议直接和二方包的提供者联系，及早替换。

二方库的工具类替换成新的接口也必须要通过单测，并对涉及的功能进行回归。



##### 自己库

作为接口或对象的提供者，废弃的类、属性、函数加上废弃的原因和替代方案。

如 RPC 订单常见接口的 `OrderCreateParam` 参数类的 JSON 类型参数：`orderItemDetail` 要替换成列表 `orderItemParams` 下面的属性类型进行替换：

```java
public class OrderCreateParam {

    /**
     * 对象详情
     * 参考示例：'[{"count":22,"name":"商品1"},{"count":33,"name":"商品2"}]'
     * <p>
     * 废弃原因：订单详情由JSON传参，改为对象传参。
     * 替代方案： {@link com.imooc.basic.deprecated.OrderCreateParam#orderItemParams}
     */
    @Deprecated
    private String orderItemDetail;

    private List<OrderItemParam> orderItemParams;

    // 其他属性
}
```

自己类的变动要通过单元测试进行验证：

```java
@Test
public void testOriginAndNew() {

    OrderCreateParam orderCreateParamOrigin = new OrderCreateParam();
    // 原始JSON属性
    orderCreateParamOrigin.setOrderItemDetail("[{\"count\":22,\"name\":\"商品1\"},{\"count\":33,\"name\":\"商品2\"}]");

    OrderCreateParam orderCreateParamNew = new OrderCreateParam();
    // 新的对象属性
    List<OrderItemParam> orderItemParamList = new ArrayList<>(2);
    OrderItemParam orderItemParam = new OrderItemParam();
    orderItemParam.setName("商品1");
    orderItemParam.setCount(22);
    orderItemParamList.add(orderItemParam);

    OrderItemParam orderItemParam2 = new OrderItemParam();
    orderItemParam2.setName("商品2");
    orderItemParam2.setCount(33);
    orderItemParamList.add(orderItemParam2);
    orderCreateParamNew.setOrderItemParams(orderItemParamList);

    Assert.assertEquals(JSON.toJSONString(orderCreateParamNew.getOrderItemParams()), orderCreateParamOrigin.getOrderItemDetail());
}
```

这里给出一个简单的模拟范例，实际业务代码中参数的接口还要进行 mock 单元测试（后续章节会有相关介绍），对应接口要根据变动传入不同的参数进行功能测试。

如如果实际开发中自己需要改动的功能涉及到废弃的类、属性、函数等，且没有详细地注释，无法获知废弃的原因和替代的方案。可以通过 IDEA 的 “annotate” 菜单，或者 “Git” - ”Show History for Selection“ 等来查看添加废弃注解的人员与之联系。避免自己错代码，如果搞明白问题且仍然不能废弃，最好能够主动将废弃的原因和替代的代码补充到注释中。

如果是三方或二方库，由于作者责任性不强或者职业素养不高，对某个接口标记废弃且没有任何注释时，我们优先在本类中寻找函数签名相似的函数。如果是开源项目或者公司内部可以拉取的项目，可以拉取该项目代码，找到该类查看提交记录，从中寻找线索。

不管是三方、二方还是自己的项目，对替换废弃的类、属性和方法等进行修改后，一定要通过单元测试去验证功能并且对接口使用的功能进行功能测试。

如果要删除废弃的属性或接口，一般先提供新的方案通知使用方修改，此时可以在将废弃的接口上加上日志，新旧接口同时运行一段时间后确认无调用再下一个版本中考虑删除接口。

如果我们能快速找到替代的方案，就可以节省很多时间；如果我们能够充分地测试，就可以平稳替换；如果我们能够介绍清楚废弃的原因，提供新的替代方案，并给出快捷的跳转方式，我们的专业程度就会提高。



#### 总结

本节的主要介绍过期类、属性、接口的正确处理姿势，包括添加废弃注解，添加废弃的原因，添加新接口的跳转等方式，还要在替换后对新接口进行测试测试。本小节还介绍了通过查看相关的优秀开源代码、使用 codota 工具来学习相关知识的方法。

下节我们将学习开发中经常碰到的又爱又恨的空指针，了解其产生的主要原因，学习如何尽可能地避免。



#### 思考题

从 github 上拉取 Spring 源码，找到 `org.springframework.util.Assert#doesNotContain(java.lang.String, java.lang.String, java.lang.String)` 方法的单元测试代码并运行。



### 空指针引发的血案



#### 前言

《手册》的第 7 页和 25 页有两段关于空指针的描述 [1](https://www.imooc.com/read/55/article/1145#fn1)：

> 【强制】Object 的 equals 方法容易抛空指针异常，应使用常量或确定有值的对象来调用 equals。
>
> 【推荐】防止 NPE，是程序员的基本修养，注意 NPE 产生的场景:
>
> 1. 返回类型为基本数据类型，return 包装数据类型的对象时，自动拆箱有可能产生 NPE。
>
> 反例:public int f () { return Integer 对象}， 如果为 null，自动解箱抛 NPE。
>
> 1. 数据库的查询结果可能为 null。
> 2. 集合里的元素即使 isNotEmpty，取出的数据元素也可能为 null。
> 3. 远程调用返回对象时，一律要求进行空指针判断，防止 NPE。
> 4. 对于 Session 中获取的数据，建议进行 NPE 检查，避免空指针。
> 5. 级联调用 obj.getA ().getB ().getC (); 一连串调用，易产生 NPE。

《手册》对空指针常见的原因和基本的避免空指针异常的方式给了介绍，非常有参考价值。

那么我们思考以下几个问题：

- 如何学习 `NullPointerException`（简称为 NPE）？
- 哪些用法可能造 NPE 相关的 BUG？
- 在业务开发中作为接口提供者和使用者如何更有效地避免空指针呢？



#### 了解空指针



##### 源码注释

前面介绍过源码是学习的一个重要途径，我们一起看看 `NullPointerException` 的源码：

```java
/**
 * Thrown when an application attempts to use {@code null} in a
 * case where an object is required. These include:
 * <ul>
 * <li>Calling the instance method of a {@code null} object.
 * <li>Accessing or modifying the field of a {@code null} object.
 * <li>Taking the length of {@code null} as if it were an array.
 * <li>Accessing or modifying the slots of {@code null} as if it
 *     were an array.
 * <li>Throwing {@code null} as if it were a {@code Throwable}
 *     value.
 * </ul>
 * <p>
 * Applications should throw instances of this class to indicate
 * other illegal uses of the {@code null} object.
 *
 * {@code NullPointerException} objects may be constructed by the
 * virtual machine as if {@linkplain Throwable#Throwable(String,
 * Throwable, boolean, boolean) suppression were disabled and/or the
 * stack trace was not writable}.
 *
 * @author  unascribed
 * @since   JDK1.0
 */
public
class NullPointerException extends RuntimeException {
    private static final long serialVersionUID = 5162710183389028792L;

    /**
     * Constructs a {@code NullPointerException} with no detail message.
     */
    public NullPointerException() {
        super();
    }

    /**
     * Constructs a {@code NullPointerException} with the specified
     * detail message.
     *
     * @param   s   the detail message.
     */
    public NullPointerException(String s) {
        super(s);
    }
}
```

源码注释给出了非常详尽地解释：

> 空指针发生的原因是应用需要一个对象时却传入了 `null`，包含以下几种情况：
>
> 1. 调用 null 对象的实例方法。
> 2. 访问或者修改 null 对象的属性。
> 3. 获取值为 null 的数组的长度。
> 4. 访问或者修改值为 null 的二维数组的列时。
> 5. 把 null 当做 Throwable 对象抛出时。

**实际编写代码时，产生空指针的原因都是这些情况或者这些情况的变种。**

《手册》中的另外一处描述

> “集合里的元素即使 isNotEmpty，取出的数据元素也可能为 `null`。”

和第 4 条非常类似。

如《手册》中的：

> “级联调用 obj.getA ().getB ().getC (); 一连串调用，易产生 NPE。”

和第 1 条很类似，因为每一层都可能得到 `null` 。

当遇到《手册》中和源码注释中所描述的这些场景时，要注意预防空指针。

另外通过读源码注释我们还得到了 “意外发现”，JVM 也可能会通过 `Throwable#Throwable(String, Throwable, boolean, boolean)` 构造函数来构造 `NullPointerException` 对象。



##### 继承体系

通过源码可以看到 NPE 继承自 `RuntimeException` 我们可以通过 IDEA 的 “Java Class Diagram” 来查看类的继承体系。

![图片描述](img/5db65d0400019f9309360449.png)
可以清晰地看到 NPE 继承自 `RuntimeException` ，另外我们选取 `NoSuchFieldException` 和 `NoSuchFieldError` 和 `NoClassDefFoundError` ，可以看到 `Throwable` 的子类型包括 `Error` 和 `Exception`, 其中 NPE 又是 `Exception` 的子类。

那么为什么 `Exception` 和 `Error` 有什么区别？ `Excption` 又分为哪些类型呢？

我们可以分别去 `java.lang.Exception` 和 `java.lang.Error` 的源码注释中寻找答案。

通过 `Exception` 的源码注释我们了解到， `Exception` 分为两类一种是非受检异常（uncheked exceptions）即 `java.lang.RuntimeException` 以及其子类；而受检异常（checked exceptions）的抛出需要再普通函数或构造方法上通过 `throws` 声明。

通过 `java.lang.Error` 的源码注释我们了解到，`Error` 代表严重的问题，不应该被程序 `try-catch`。编译时异常检测时， `Error` 也被视为不可检异常（uncheked exceptions）。

大家可以在 IDEA 中分别查看 `Exception` 和 `Error` 的子类，了解自己开发中常遇到的异常都属于哪个分类。

我们还可以通过《JLS》[2](https://www.imooc.com/read/55/article/1145#fn2) 第 11 章 [Exceptions](https://docs.oracle.com/javase/specs/jls/se8/html/jls-11.html) 对异常进行学习。

其中在异常的类型这里，讲到：

> 不可检异常（ **unchecked exception**）包括运行时异常和 error 类。
>
> 可检异常（ **checked exception** ）不属于不可检异常的所有异常都是可检异常。除 RuntimeException 和其子类，以及 Error 类以及其子类外的其他 Throwable 的子类。

![图片描述](img/5db65cf800014cea06860267.png)
还有更多关于异常的详细描述，，包括异常的原因、异步异常、异常的编译时检查等，大家可以自己进一步学习。



#### 空指针引发的血案



##### 最常见的错误姿势

```java
 @Test
    public void test() {
        Assertions.assertThrows(NullPointerException.class, () -> {
            List<UserDTO> users = new ArrayList<>();
            users.add(new UserDTO(1L, 3));
            users.add(new UserDTO(2L, null));
            users.add(new UserDTO(3L, 3));
            send(users);
        });

    }

    // 第 1 处
    private void send(List<UserDTO> users) {
        for (UserDTO userDto : users) {
            doSend(userDto);
        }
    }

    private static final Integer SOME_TYPE = 2;

    private void doSend(UserDTO userDTO) {
        String target = "default";
        // 第 2 处
        if (!userDTO.getType().equals(SOME_TYPE)) {
            target = getTarget(userDTO.getType());
        }
        System.out.println(String.format("userNo:%s, 发送到%s成功", userDTO, target));
    }

    private String getTarget(Integer type) {
        return type + "号基地";
    }
```

在第 1 处，如果集合为 `null` 则会抛空指针；

在第 2 处，如果 `type` 属性为 `null` 则会抛空指针异常，导致后续都发送失败。

大家看这个例子觉得很简单，看到输入的参数有 `null` 本能地就会考虑空指针问题，但是自己写代码时你并不知道上游是否会有 `null`。



##### 无结果仍返回对象

实际开发中有些同学会有一些非常 “个性” 的写法。

为了避免空指针或避免检查到 null 参数抛异常，直接返回一个空参构造函数创建的对象。

类似下面的做法：

```java
/**
 * 根据订单编号查询订单
 *
 * @param orderNo 订单编号
 * @return 订单
 */
public Order getByOrderNo(String orderNo) {

    if (StringUtils.isEmpty(orderNo)) {
        return new Order();
    }
    // 查询order
    return doGetByOrderNo(orderNo);
}
```

由于常见的单个数据的查询接口，参数检查不符时会抛异常或者返回 `null`。 极少有上述的写法，因此调用方的惯例是判断结果不为 `null` 就使用其中的属性。

这个哥们这么写之后，上层判断返回值不为 `null` , 上层就放心大胆得调用实例函数，导致线上报空指针，就造成了线上 BUG。



##### 新增 @NonNull 属性反序列化的 BUG

假如有一个订单更新的 RPC 接口，该接口有一个 `OrderUpdateParam` 参数，之前有两个属性一个是 `id` 一个是 `name` 。在某个需求时，新增了一个 extra 属性，且该字段一定不能为 `null` 。

采用 lombok 的 `@NonNull` 注解来避免空指针：

```java
import lombok.Data;
import lombok.NonNull;

import java.io.Serializable;

@Data
public class OrderUpdateParam implements Serializable {
    private static final long serialVersionUID = 3240762365557530541L;

    private Long id;

    private String name;

     // 其它属性
  
    // 新增的属性
    @NonNull
    private String extra;
}
```

上线后导致没有使用最新 jar 包的服务对该接口的 RPC 调用报错。

我们来分析一下原因，在 IDEA 的 target - classes 目录下找到 DEMO 编译后的 class 文件，IDEA 会自动帮我们反编译：

```java
public class OrderUpdateParam implements Serializable {
    private static final long serialVersionUID = 3240762365557530541L;
    private Long id;
    private String name;
    @NonNull
    private String extra;

    public OrderUpdateParam(@NonNull final String extra) {
        if (extra == null) {
            throw new NullPointerException("extra is marked non-null but is null");
        } else {
            this.extra = extra;
        }
    }

    @NonNull
    public String getExtra() {
        return this.extra;
    }
    public void setExtra(@NonNull final String extra) {
        if (extra == null) {
            throw new NullPointerException("extra is marked non-null but is null");
        } else {
            this.extra = extra;
        }
    }
  // 其他代码

}
```

我们还可以使用反编译工具：[JD-GUI](http://java-decompiler.github.io/) 对编译后的 class 文件进行反编译，查看源码。

由于调用方调用的是不含 `extra` 属性的 jar 包，并且序列化编号是一致的，反序列化时会抛出 NPE。

```java
Caused by: java.lang.NullPointerException: extra

        at com.xxx.OrderUpdateParam.<init>(OrderUpdateParam.java:21)
```

RPC 参数新增 lombok 的 `@NonNull` 注解时，要考虑调用方是否及时更新 jar 包，避免出现空指针。



##### 自动拆箱导致空指针

前面章节讲到了对象转换，如果我们下面的 `GoodCreateDTO` 是我们自己服务的对象， 而 `GoodCreateParam` 是我们调用服务的参数对象。

```java
@Data
public class GoodCreateDTO {
    private String title;

    private Long price;

    private Long count;
}

@Data
public class GoodCreateParam implements Serializable {

    private static final long serialVersionUID = -560222124628416274L;
    private String title;

    private long price;

    private long count;
}
```

其中 `GoodCreateDTO` 的 `count` 属性在我们系统中是非必传参数，本系统可能为 `null`。

如果我们没有拉取源码的习惯，直接通过前面的转换工具类去转换。

我们潜意识会认为外部接口的对象类型也都是包装类型，这时候很容易因为转换出现 NPE 而导致线上 BUG。

```java
public class GoodCreateConverter {

    public static GoodCreateParam convertToParam(GoodCreateDTO goodCreateDTO) {
        if (goodCreateDTO == null) {
            return null;
        }
        GoodCreateParam goodCreateParam = new GoodCreateParam();
        goodCreateParam.setTitle(goodCreateDTO.getTitle());
        goodCreateParam.setPrice(goodCreateDTO.getPrice());
        goodCreateParam.setCount(goodCreateDTO.getCount());
        return goodCreateParam;
    }
}
```

当转换器执行到 `goodCreateParam.setCount(goodCreateDTO.getCount());` 会自动拆箱会报空指针。

当 `GoodCreateDTO` 的 `count` 属性为 `null` 时，自动拆箱将报空指针。

**再看一个花样踩坑的例子**：

我们作为使用方调用如下的二方服务接口：

```java
public Boolean someRemoteCall();
```

然后自以为对方肯定会返回 `TRUE` 或 `FALSE`，然后直接拿来作为判断条件或者转为基本类型，如果返回的是 `null`，则会报空指针异常：

```java
if (someRemoteCall()) {
           // 业务代码
 }
```

大家看示例的时候可能认为这种情况很简单，自己开发的时候肯定会注意，但是往往事实并非如此。

希望大家可以掌握常见的可能发生空指针场景，在开发是注意预防。



##### 分批调用合并结果时空指针

大家再看下面这个经典的例子。

因为某些批量查询的二方接口在数据较大时容易超时，因此可以分为小批次调用。

下面封装一个将 `List` 数据拆分成每 `size` 个一批数据，去调用 `function` RPC 接口，然后将结果合并。

```java
  public static <T, V> List<V> partitionCallList(List<T> dataList, int size, Function<List<T>, List<V>> function) {

        if (CollectionUtils.isEmpty(dataList)) {
            return new ArrayList<>(0);
        }
        Preconditions.checkArgument(size > 0, "size 必须大于0");

        return Lists.partition(dataList, size)
                .stream()
                .map(function)
                .reduce(new ArrayList<>(),
                        (resultList1, resultList2) -> {
                            resultList1.addAll(resultList2);
                            return resultList1;
                        });


    }
```

看着挺对，没啥问题，其实则不然。

设想一下，如果某一个批次请求无数据，不是返回空集合而是 null，会怎样？

很不幸，又一个空指针异常向你飞来 …

此时**要根据具体业务场景来判断如何处理这里可能产生的空指针异常**。

如果在某个场景中，返回值为 null 是一定不允许的行为，可以在 function 函数中对结果进行检查，如果结果为 null，可抛异常。

如果是允许的，在调用 map 后，可以过滤 null :

```
// 省略前面代码
.map(function)
.filter(Objects::nonNull)
// 省略后续代码
```



#### 预防空指针的一些方法

`NPE` 造成的线上 BUG 还有很多种形式，如何预防空指针很重要。

下面将介绍几种预防 NPE 的一些常见方法：

![图片描述](img/5db65cce0001e2aa16700938.png)



##### 接口提供者角度



###### 返回空集合

如果参数不符合要求直接返回空集合，底层的函数也使用一致的方式：

```java
public List<Order> getByOrderName(String name) {
    if (StringUtils.isNotEmpty(name)) {
        return doGetByOrderName(name);
    }
    return Collections.emptyList();
}
```



###### 使用 Optional

`Optional` 是 Java 8 引入的特性，返回一个 `Optional` 则明确告诉使用者结果可能为空：

```java
public Optional<Order> getByOrderId(Long orderId) {
    return Optional.ofNullable(doGetByOrderId(orderId));
}
```

如果大家感兴趣可以进入 `Optional` 的源码，结合前面介绍的 `codota` 工具进行深入学习，也可以结合《Java 8 实战》的相关章节进行学习。



###### 使用空对象设计模式

该设计模式为了解决 NPE 产生原因的第 1 条 “调用 `null` 对象的实例方法”。

在编写业务代码时为了避免 `NPE` 经常需要先判空再执行实例方法：

```java
public void doSomeOperation(Operation operation) {
    int a = 5;
    int b = 6;
    if (operation != null) {
        operation.execute(a, b);
    }
}
```

《设计模式之禅》（第二版）554 页在拓展篇讲述了 “空对象模式”。

可以构造一个 `NullXXX` 类拓展自某个接口， 这样这个接口需要为 `null` 时，直接返回该对象即可：

```java
public class NullOperation implements Operation {

    @Override
    public void execute(int a, int b) {
        // do nothing
    }
}
```

这样上面的判空操作就不再有必要， 因为我们在需要出现 `null` 的地方都统一返回 `NullOperation`，而且对应的对象方法都是有的：

```java
public void doSomeOperation(Operation operation) {
    int a = 5;
    int b = 6;
    operation.execute(a, b);
}
```



##### 接口使用者角度

讲完了接口的编写者该怎么做，我们讲讲接口的使用者该如何避免 `NPE` 。



###### null 检查

正如《代码简洁之道》第 7.8 节 “别传 null 值” 中所要表达的意义：

> 可以进行参数检查，对不满足的条件抛出异常。

直接在使用前对不能为 `null` 的和不满足业务要求的条件进行检查，是一种最简单最常见的做法。

通过防御性参数检测，可以极大降低出错的概率，提高程序的健壮性：

```java
    @Override
    public void updateOrder(OrderUpdateParam orderUpdateParam) {
        checkUpdateParam(orderUpdateParam);
        doUpdate(orderUpdateParam);
    }

    private void checkUpdateParam(OrderUpdateParam orderUpdateParam) {
        if (orderUpdateParam == null) {
            throw new IllegalArgumentException("参数不能为空");
        }
        Long id = orderUpdateParam.getId();
        String name = orderUpdateParam.getName();
        if (id == null) {
            throw new IllegalArgumentException("id不能为空");
        }
        if (name == null) {
            throw new IllegalArgumentException("name不能为空");
        }
    }
```

JDK 和各种开源框架中可以找到很多这种模式，`java.util.concurrent.ThreadPoolExecutor#execute` 就是采用这种模式。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
     // 其他代码
     }
```

以及 `org.springframework.context.support.AbstractApplicationContext#assertBeanFactoryActive`

```java
protected void assertBeanFactoryActive() {
   if (!this.active.get()) {
      if (this.closed.get()) {
         throw new IllegalStateException(getDisplayName() + " has been closed already");
      }
      else {
         throw new IllegalStateException(getDisplayName() + " has not been refreshed yet");
      }
   }
}
```



###### 使用 Objects

可以使用 Java 7 引入的 Objects 类，来简化判空抛出空指针的代码。

使用方法如下：

```java
private void checkUpdateParam2(OrderUpdateParam orderUpdateParam) {
    Objects.requireNonNull(orderUpdateParam);
    Objects.requireNonNull(orderUpdateParam.getId());
    Objects.requireNonNull(orderUpdateParam.getName());
}
```

原理很简单，我们看下源码；

```java
public static <T> T requireNonNull(T obj) {
    if (obj == null)
        throw new NullPointerException();
    return obj;
}
```



###### 使用 commons 包

我们可以使用 commons-lang3 或者 commons-collections4 等常用的工具类辅助我们判空。

**使用字符串工具类：org.apache.commons.lang3.StringUtils**

```java
public void doSomething(String param) {
    if (StringUtils.isNotEmpty(param)) {
        // 使用param参数
    }
}
```

**使用校验工具类：org.apache.commons.lang3.Validate**

```java
public static void doSomething(Object param) {
    Validate.notNull(param,"param must not null");
}
public static void doSomething2(List<String> parms) {
    Validate.notEmpty(parms);
}
```

该校验工具类支持多种类型的校验，支持自定义提示文本等。

前面已经介绍了读源码是最好的学习方式之一，这里我们看下底层的源码：

```java
public static <T extends Collection<?>> T notEmpty(final T collection, final String message, final Object... values) {
    if (collection == null) {
        throw new NullPointerException(String.format(message, values));
    }
    if (collection.isEmpty()) {
        throw new IllegalArgumentException(String.format(message, values));
    }
    return collection;
}
```

该如果集合对象为 null 则会抛空 `NullPointerException` 如果集合为空则抛出 `IllegalArgumentException`。

通过源码我们还可以了解到更多的校验函数。



##### 使用集合工具类：`org.apache.commons.collections4.CollectionUtils`

```java
public void doSomething(List<String> params) {
    if (CollectionUtils.isNotEmpty(params)) {
        // 使用params
    }
}
```



##### 使用 guava 包

可以使用 guava 包的 `com.google.common.base.Preconditions` 前置条件检测类。

同样看源码，源码给出了一个范例。原始代码如下：

```java
public static double sqrt(double value) {
    if (value < 0) {
        throw new IllegalArgumentException("input is negative: " + value);
    }
    // calculate square root
}
```

使用 `Preconditions` 后，代码可以简化为：

```java
 public static double sqrt(double value) {
   checkArgument(value >= 0, "input is negative: %s", value);
   // calculate square root
 }
 
```

Spring 源码里很多地方可以找到类似的用法，下面是其中一个例子：

```
org.springframework.context.annotation.AnnotationConfigApplicationContext#register
public void register(Class<?>... annotatedClasses) {
    Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
    this.reader.register(annotatedClasses);
}
org.springframework.util.Assert#notEmpty(java.lang.Object[], java.lang.String)
public static void notEmpty(Object[] array, String message) {
    if (ObjectUtils.isEmpty(array)) {
        throw new IllegalArgumentException(message);
    }
}
```

虽然使用的具体工具类不一样，核心的思想都是一致的。



##### 自动化 API

###### 使用 lombok 的 `@Nonnull` 注解

```java
 public void doSomething5(@NonNull String param) {
      // 使用param
      proccess(param);
 }
```

查看编译后的代码：

```java
 public void doSomething5(@NonNull String param) {
      if (param == null) {
          throw new NullPointerException("param is marked non-null but is null");
      } else {
          this.proccess(param);
      }
  }
```

###### 使用 IntelliJ IDEA 提供的 @NotNull 和 @Nullable 注解

maven 依赖如下：

```xml
<!-- https://mvnrepository.com/artifact/org.jetbrains/annotations -->
<dependency>
    <groupId>org.jetbrains</groupId>
    <artifactId>annotations</artifactId>
    <version>17.0.0</version>
</dependency>
```

@NotNull 在参数上的用法和上面的例子非常相似。

```java
public static void doSomething(@NotNull String param) {
    // 使用param
    proccess(param);
}
```

我们可以去该注解的源码 `org.jetbrains.annotations.NotNull#exception` 里查看更多细节，大家也可以使用 IDEA 插件或者前面介绍的 JD-GUI 来查看编译后的 class 文件，去了解 @NotNull 注解的作用。



#### 总结

本节主要讲述空指针的含义，空指针常见的中枪姿势，以及如何避免空指针异常。下一节将为你揭秘 当 switch 遇到空指针，又会发生什么奇妙的事情。



### 当switch遇到空指针



#### 1. 前言

《手册》的第 18 页有关于 `switch` 的规约：

> 【强制】当 switch 括号内的变量类型为 String 并且此变量为外部参数时，必须先进行 null
> 判断。[1](https://www.imooc.com/read/55/article/1146#fn1)

在《手册》中，该规约下面还给出了一段反例（此处略）。

最近很火的一篇名为《悬赏征集！5 道题征集代码界前 3% 的超级王者》[2](https://www.imooc.com/read/55/article/1146#fn2) 的文章，也给出了类似的一段代码：

```java
public class SwitchTest {
    public static void main(String[] args) {
        String param = null;
        switch (param) {
            case "null":
                System.out.println("null");
                break;
            default:
                System.out.println("default");
        }
    }
}
```

该文章给出的问题是：“上面这段程序输出的结果是什么？”。

其实，想知道答案很容易，运行一下程序答案就出来了。

**但是如果浅尝辄止，我们就丧失了一次难得的学习机会**，不像是一名优秀程序猿的作风。

我们还需要思考下面几个问题：

- `switch` 除了 `String` 还支持哪种类型？
- 为什么《手册》规定字符串类型参数要先进行 null 判断？
- 为什么可能会抛出异常？
- 该如何分析这类问题呢？

本节将对上述问题进行分析。



#### 2. 问题分析



##### 2.1 源码大法

按照我们一贯的风格，我们应该先上 “源码大法”，但是 `switch` 是关键字，无法进入 JDK 源码中查看学习，因此我们暂时放弃通过源码或源码注释来分析解决的手段。



##### 2.2 官方文档

我们去官方文档 JLS[3](https://www.imooc.com/read/55/article/1146#fn3) 查看 `swtich` 语句[相关描述](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.11)。

> switch 的表达式必须是 char, byte, short, int, Character, Byte, Short, Integer, String, 或者 enum 类型，否则会发生编译错误
>
> switch 语句必须满足以下条件，否则会出现编译错误：
>
> - 与 switch 语句关联的每个 case 都必须和 switch 的表达式的类型一致；
> - 如果 switch 表达式是枚举类型，case 常量也必须是枚举类型；
> - 不允许同一个 switch 的两个 case 常量的值相同；
> - 和 switch 语句关联的常量不能为 null ；
> - 一个 switch 语句最多有一个 default 标签。

![图片描述](img/5dc28adc000186df15660988.png)

我们了解到 switch 语句支持的类型，以及会出现编译错误的原因。

我们看到关键的一句话：

> When the switch statement is executed, first the Expression is evaluated. If the Expression evaluates to null, a NullPointerException is thrown and the entire switch statement completes abruptly for that reason.
>
> switch 语句执行的时候，首先将执行 switch 的表达式。如果表达式为 null, 则会抛出 NullPointerException，整个 switch 语句的执行将被中断。

这里的表达式就是我们的参数，前言中该参数的值为 `null`, 因此答案就显而易见了：结果会抛出异常，而且是前面章节讲到的 `NullPointerException`。

另外从 JVM[4](https://www.imooc.com/read/55/article/1146#fn4) [3.10 节 “Compiling Switches”](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html#jvms-3.10) ，我们学习到：

> 编译器使用 tableswitch 和 lookupswitch 指令生成 switch 语句的编译代码。tablesswtich 语句用于表示 swtich 结构的 case 语句块，它可以地从索引表中确定 case 语句块的分支偏移量。当 switch 语句的条件值不能对应索引表的任何一个 case 语句块的偏移量时就会用到 default 语句。
>
> Java 虚拟机的 tableswitch 和 lookupswitch 指令只能支持 int 类型的条件值。如果 swich 中使用其他类型的值，那么就必须转化为 int 类型。
>
> 当 switch 语句中的 case 分支条件比较稀疏时， tableswtich 指令的空间利用率较低。 可以使用 lookupswitch 指令来取代。
>
> lookupswitch 指令的索引表项由 int 类型的键（来自于 case 语句后的数值）和对应目标语句的偏移量构成。 当 lookcupswitch 指令执行时， switch 语句的条件值将和索引表中的键进行比对，如果某个键和条件的值相符，那么将转移到这个键对应的分支偏移量的代码行处开始执行，如果没有符合的键值，则执行 default 分支。

因此我们可以推测出，表达式会将 String 的参数转成 int 类型的值和 case 进行比对。

我们去 `String` 源码中寻找可以将字符串转 int 的函数，发现 `hashCode()` 可能是最佳的选择之一（后面会印证）。

因此空指针出现的根源在于：**虚拟机为了实现 switch 的语法，将参数表达式转换成 int。而这里的参数为 null， 从而造成了空指针异常**。

通过官方文档的阅读，我们对 switch 有了一个相对深入的了解。



##### 2.3 Java 反汇编大法

如何印证官方文档的描述？如何进一步分析呢？

按照惯例我们用反汇编大法。



###### 2.3.1 switch 举例

我们先看一个正常的示例：

```java
public static void main(String[] args) {
        String param = "t";
        switch (param) {
            case "a":
                System.out.println("a");
                break;
            case "b":
                System.out.println("b");
                break;
            case "c":
                System.out.println("c");
                break;
            default:
                System.out.println("default");
        }
```

先进入到代码目录，对类文件进行编译：

```
javac SwitchTest2.java
```

然后反汇编的代码如下：

```
javap -c SwitchTest2
```

前方高能预警，先稳住，不要怕，不要方，后面会给出解释并给出简化版：

```java
Compiled from "SwitchTest2.java"
public class com.imooc.basic.learn_switch.SwitchTest2 {
  public com.imooc.basic.learn_switch.SwitchTest2();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String t
       2: astore_1
       3: aload_1
       4: astore_2
       5: iconst_m1
       6: istore_3
       7: aload_2
       8: invokevirtual #3                  // Method java/lang/String.hashCode:()I
      11: tableswitch   { // 97 to 99
                    97: 36
                    98: 50
                    99: 64
               default: 75
          }
      36: aload_2
      37: ldc           #4                  // String a
      39: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      42: ifeq          75
      45: iconst_0
      46: istore_3
      47: goto          75
      50: aload_2
      51: ldc           #6                  // String b
      53: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      56: ifeq          75
      59: iconst_1
      60: istore_3
      61: goto          75
      64: aload_2
      65: ldc           #7                  // String c
      67: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      70: ifeq          75
      73: iconst_2
      74: istore_3
      75: iload_3
      76: tableswitch   { // 0 to 2
                     0: 104
                     1: 115
                     2: 126
               default: 137
          }
     104: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     107: ldc           #4                  // String a
     109: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     112: goto          145
     115: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     118: ldc           #6                  // String b
     120: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     123: goto          145
     126: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     129: ldc           #7                  // String c
     131: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     134: goto          145
     137: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     140: ldc           #10                 // String default
     142: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     145: return
}
```

首先介绍一个简单的背景知识：

> 字符 a 的 ASCII 码为 97, b 为 98，c 为 99 （我们发现常见英文字母的哈希值为其 ASCII 码）。

tableswitch 后面的注释显示 case 的哈希值的范围是 97 到 99。

我们讲解核心代码，先看偏移为 8 的指令，调用了参数的 `hashCode()` 函数来获取字符串 "t" 的哈希值。

```java
tableswitch   { // 97 to 99
           97: 36
           98: 50
           99: 64
           default: 75
      }
```

接下来我们看偏移为 11 的指令处： tableswitch 是跳转引用列表， 如果值小于其中的最小值或者大于其中的最大值，跳转到 `default` 语句。

> 其中 97 为键，36 为对应的目标语句偏移量。

hashCode 和 tableswitch 的键相等，则跳转到对应的目标偏移量，t 的哈希值为 116，大于条件的最大值 99，因此跳转到 `default` 对应的语句行（即偏移量为 75 的指令处执行）。

从 36 到 74 行，根据哈希值相等跳转到判断是否相等的指令。

然后调用 `java.lang.String#equals` 判断 switch 的字符串是否和对应的 case 的字符串相等。

如果相等则分别根据第几个条件得到条件的索引，然后每个索引对应下一个指定的代码行数。

default 语句对应 137 行，打印 “default” 字符串，然后执行 145 行 `return` 命令返回。

然后再通过 tableswitch 判断执行哪一行打印语句。

**因此整个流程是先计算字符串参数的哈希值，判断哈希值的范围，然后哈希值相等再判断对象是否相等，然后执行对应的代码块。**



###### 2.3.2 分析问题

经过前面的学习我们对 String 为参数的 switch 语句的执行流程有了初步认识。

我们反汇编开篇的示例，得到如下代码：

```java
Compiled from "SwitchTest.java"
public class com.imooc.basic.learn_switch.SwitchTest {
  public com.imooc.basic.learn_switch.SwitchTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: aconst_null
       1: astore_1
       2: aload_1
       3: astore_2
       4: iconst_m1
       5: istore_3
       6: aload_2
       7: invokevirtual #2                  // Method java/lang/String.hashCode:()I
      10: lookupswitch  { // 1
               3392903: 28
               default: 39
          }
      28: aload_2
      29: ldc           #3                  // String null
      31: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      34: ifeq          39
      37: iconst_0
      38: istore_3
      39: iload_3
      40: lookupswitch  { // 1
                     0: 60
               default: 71
          }
      60: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      63: ldc           #3                  // String null
      65: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      68: goto          79
      71: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      74: ldc           #7                  // String default
      76: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      79: return
}
```

**猜想和验证是学习的最佳方式之一**，我们通过猜想来提取知识，通过验证来核实自己的猜想是否正确。

**猜想 1**：根据上面的分析我们可以 “猜想”：3392903 应该是 "null" 字符串的哈希值。

我们可以打印其哈希值去印证：`System.out.println(("null").hashCode());` ，也可以通过编写单元测试来断言，还可以通过调试来执行表达式等方式查看。

在调试模式下，在变量选项卡上右键，选择 “Evaluate Expression…” ，填写想执行想计算的表达式即可：

![图片描述](img/5dc2a76b0001c49814441042.png)

我们将上面的字节码的逻辑反向 “翻译” 成 java 代码大致如下：

```java
String param = null;
int hashCode = param.hashCode();
if(hashCode == ("null").hashCode() && param.equals("null")){ 
   System.out.println("null");
}else{
   System.out.println("default");
}
```

对应流程图如下：

![图片描述](img/5dc28a9c0001ee3e10981172.png)

因此空指针的原因就一目了然了。

回忆一下空指针的小节讲到的：

> 空指针异常发生的原因之一：“调用 null 对象的实例方法。”。
>
> 以及 “JVM 也可能会通过 `Throwable#Throwable(String, Throwable, boolean, boolean)` 构造函数来构造 `NullPointerException` 对象。”

此处字节码执行时调用了 `null` 的 `hashCode` 方法，虚拟机可以通过上面的函数构造 NPE 并抛出。

**那么将字符串通过 hashCode 函数转为整型和 case 条件对比后，为什么还需要 equals 再次判断呢？**

这就要回到 hashCode 函数的本质，即将不同的对象（不定长）映射到整数范围（定长）, 而且 java 的 hashCode 函数和 equals 函数默认约定：同一个对象的 hashCode 一定相等， 即 hashCode 不等的对象一定不是同一个对象。

详情参见 `java.lang.Object#hashCode` 和 `java.lang.Object#equals` 的注释。

通过这一特性，可以快速判断对象是否有可能相当，避免不必要的比较。

**另外我们还可以猜想如何提高比较的效率？**

**猜想 2：** 如果编译期能够将 lookupswitch 按照 hash 值升序排序，则运行时就可讲参数的 hash 值（最小）先和第一个和除 default 外的倒数第一个 hash 值（最大）比较，不在这个范围直接走 default 语句即可，在这个范围就可以使用使用二分查找法，将时间复杂度降低到 O (logn) ，从而大大提高效率。

大家可以通过读 jvms 甚至读虚拟机代码去核实和验证上述猜想。

另外，**虽然有些哈希函数设计的比较优良，能够尽可能避免 hash 冲突，但是对象的数量是 “无限” 的，整数的范围是 “有限” 的，将无限的对象映射到有限的范围，必然会产生冲突。**

因此通过上述反汇编代码可以看出：

switch 表达式会先计算字符串的 hashCode （main 函数偏移为 7 处代码），然后根据 hashCode 是否相等快速判断是否要走到某个 case（见 lookupswith），如果不满足，直接执行到 default （main 函数偏移为 39 处代码）；如果满足，则跳转到对应 case 的代码（见 main 函数偏移为 28 之后的代码）再通过 equals 判断值是否相等，来避免 hash 冲突时 case 被误执行。

**这种先判断 hash 值是否相等（有可能是同一个对象 / 两个对象有可能相等）再通过 equals 比较 “对象是否相等” 的做法，在 Java 的很多 JDK 源码中和其他框架中非常常见**。



#### 3. 总结

本节我们结合一个简单的案例 和 jvms， 学习了 switch 的基本原理，分析了示例代码产生空指针的原因。本节还介绍了一个简单的调试技巧，以及 “猜想和验证” 的学习方式，希望大家在后面的学习和工作中多加实践。

下一节我们将深入学习枚举并介绍其高级用法。



#### 4. 课后题

下面的代码结果是啥呢？

```java
public class SwitchTest {
    public static void main(String[] args) {
        String param = null;
        switch (param="null") {
            case "null":
                System.out.println("null");
                break;
            default:
                System.out.println("default");
        }
    }
```

大家可以通过今天学习的知识，自己去实战分析这个问题。



### 枚举类的正确学习方式



#### 1. 前言

《手册》第 3 、4 、39 页中有几段关于枚举类型的描述 [1](https://www.imooc.com/read/55/article/1147#fn1) ：

> 【参考】枚举类名带上 Enum 后缀，枚举成员名称需要全大写，单词间用下划线隔开。
> 说明：枚举其实就是特殊的类，域成员均为常量，且构造方法被默认强制是私有。
>
> 【推荐】如果变量值仅在一个固定范围内变化用 enum 类型来定义。
>
> 【强制】二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用 枚举类型或者包含枚举类型的 POJO 对象。

大多数 Java 程序员对枚举类型一知半解，大多数程序员对枚举的用法都非常简单。

本小节主要解决以下几个问题：

- 那么枚举类究竟是怎样的？
- 默认的构造方法为何是私有的？
- 为什么接口不要返回枚举类型。
- 枚举类还有哪些高级用法？



#### 2. 学习枚举类



##### 2.1 勿忘初心

我们学习一个框架，学习一个语言特性时，可以思考一下这个框架和语言特性出现的原因。

枚举一般用来表示一组相同类型的常量，比如月份、星期、颜色等。

枚举的主要使用场景是，当需要一组固定的常量，并且编译时成员就已能确定时就应该使用枚举。[2](https://www.imooc.com/read/55/article/1147#fn2)

因此枚举类型没必要多例，如果能够保证单例，则可以减少内存开销。

另外枚举为数值提供了命名，更容易理解，而且枚举更加安全，功能更加强大。



##### 2.2 官方文档法

前面介绍过，优先通过官方文档来学习 Java 的语言特性。

JLS [8.9 节 Enum Types](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.9) 对枚举类型进行了详细地介绍 [3](https://www.imooc.com/read/55/article/1147#fn3)。主要有以下几个要点：

> 如果枚举类如果被 abstract 或 final 修饰，枚举如果常量重复，如果尝试实例化枚举类型都会有编译错误。
>
> 枚举类除声明的枚举常量没有其他实例。
>
> 枚举类型的 E 是 Enum 的直接子类。

那么 Java 是如何保证除了定义的枚举常量外没有其他实例呢？

从手册中我们可以找到原因：

- Enum 的 clone 方法被 final 修饰，保证 enum 常量不会被克隆。
- 禁止对枚举类型的反射。
- 序列化机制保证反序列化时枚举类型不允许构造多个相同实例。

通过这些提示，我们就明白为何枚举类的构造函数是私有的，

文档中还介绍了枚举的成员，枚举的迭代，枚举类型作为 switch 的条件，带抽象函数的枚举常量等。



##### 2.3 Java 反汇编

我们选取 JLS 中的一个代码片段：

```java
public enum CoinEnum {
    PENNY(1), NICKEL(5), DIME(10), QUARTER(25);

    CoinEnum(int value) {
        this.value = value;
    }

    private final int value;
    public int value() { return value; }
}
```

先编译： `javac CoinEnum.java`

然后再反汇编：`javap -c CoinEnum`

得到下面的反汇编后的代码：

```java
public final class com.imooc.basic.learn_enum.CoinEnum extends java.lang.Enum<com.imooc.basic.learn_enum.CoinEnum> {
  public static final com.imooc.basic.learn_enum.CoinEnum PENNY;

  public static final com.imooc.basic.learn_enum.CoinEnum NICKEL;

  public static final com.imooc.basic.learn_enum.CoinEnum DIME;

  public static final com.imooc.basic.learn_enum.CoinEnum QUARTER;

  // 第 1 处代码
  public static com.imooc.basic.learn_enum.CoinEnum[] values();
    Code:
       0: getstatic     #1                  // Field $VALUES:[Lcom/imooc/basic/learn_enum/CoinEnum;
       3: invokevirtual #2                  // Method "[Lcom/imooc/basic/learn_enum/CoinEnum;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[Lcom/imooc/basic/learn_enum/CoinEnum;"
       9: areturn

   // 第 2 处代码
  public static com.imooc.basic.learn_enum.CoinEnum valueOf(java.lang.String);
    Code:
       0: ldc           #4                  // class com/imooc/basic/learn_enum/CoinEnum
       2: aload_0
       3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
       6: checkcast     #4                  // class com/imooc/basic/learn_enum/CoinEnum
       9: areturn

  public int value();
    Code:
       0: aload_0
       1: getfield      #7                  // Field value:I
       4: ireturn

  static {};
    Code:
       0: new           #4                  // class com/imooc/basic/learn_enum/CoinEnum
       3: dup
       4: ldc           #8                  // String PENNY
       6: iconst_0
       7: iconst_1
       8: invokespecial #9                  // Method "<init>":(Ljava/lang/String;II)V
      11: putstatic     #10                 // Field PENNY:Lcom/imooc/basic/learn_enum/CoinEnum;
      14: new           #4                  // class com/imooc/basic/learn_enum/CoinEnum
      17: dup
      18: ldc           #11                 // String NICKEL
      20: iconst_1
      21: iconst_5
      22: invokespecial #9                  // Method "<init>":(Ljava/lang/String;II)V
      25: putstatic     #12                 // Field NICKEL:Lcom/imooc/basic/learn_enum/CoinEnum;
      28: new           #4                  // class com/imooc/basic/learn_enum/CoinEnum
      31: dup
      32: ldc           #13                 // String DIME
      34: iconst_2
      35: bipush        10
      37: invokespecial #9                  // Method "<init>":(Ljava/lang/String;II)V
      40: putstatic     #14                 // Field DIME:Lcom/imooc/basic/learn_enum/CoinEnum;
      43: new           #4                  // class com/imooc/basic/learn_enum/CoinEnum
      46: dup
      47: ldc           #15                 // String QUARTER
      49: iconst_3
      50: bipush        25
      52: invokespecial #9                  // Method "<init>":(Ljava/lang/String;II)V
      55: putstatic     #16                 // Field QUARTER:Lcom/imooc/basic/learn_enum/CoinEnum;
      58: iconst_4
      59: anewarray     #4                  // class com/imooc/basic/learn_enum/CoinEnum
      62: dup
      63: iconst_0
      64: getstatic     #10                 // Field PENNY:Lcom/imooc/basic/learn_enum/CoinEnum;
      67: aastore
      68: dup
      69: iconst_1
      70: getstatic     #12                 // Field NICKEL:Lcom/imooc/basic/learn_enum/CoinEnum;
      73: aastore
      74: dup
      75: iconst_2
      76: getstatic     #14                 // Field DIME:Lcom/imooc/basic/learn_enum/CoinEnum;
      79: aastore
      80: dup
      81: iconst_3
      82: getstatic     #16                 // Field QUARTER:Lcom/imooc/basic/learn_enum/CoinEnum;
      85: aastore
      86: putstatic     #1                  // Field $VALUES:[Lcom/imooc/basic/learn_enum/CoinEnum;
      89: return
}
```

通过开头位置的继承关系 `com.imooc.basic.learn_enum.Coin extends java.lang.Enum`，验证了官方手册描述的 “枚举类型的 E 是 Enum 的直接子类。” 的说法。

我们还看到枚举类编译后被被自动加上 `final` 关键字。

枚举常量也会被加上 `public static final` 修饰。

另外我们还注意到和源码相比多了两个函数：

其中一个为：`public static com.imooc.basic.learn_enum.CoinEnum valueOf(java.lang.String);` （见 “第 2 处代码” ）

```java
// 第 2 处代码
  public static com.imooc.basic.learn_enum.CoinEnum valueOf(java.lang.String);
    Code:
       0: ldc           #4                  // class com/imooc/basic/learn_enum/CoinEnum
       2: aload_0
       3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
       6: checkcast     #4                  // class com/imooc/basic/learn_enum/CoinEnum
       9: areturn
```

**这是怎么回事？干嘛用的呢？**

通过第 2 处代码的 code 偏移为 3 处的代码，我们可以看出调用了 `java.lang.Enum#valueOf` 函数。

我们直接找到该函数的源码：

```java
/**
 * Returns the enum constant of the specified enum type with the
 * specified name.  The name must match exactly an identifier used
 * to declare an enum constant in this type.  (Extraneous whitespace
 * characters are not permitted.)
 *
 * <p>Note that for a particular enum type {@code T}, the
 * implicitly declared {@code public static T valueOf(String)}
 * method on that enum may be used instead of this method to map
 * from a name to the corresponding enum constant.  All the
 * constants of an enum type can be obtained by calling the
 * implicit {@code public static T[] values()} method of that
 * type.
 *
 * @param <T> The enum type whose constant is to be returned
 * @param enumType the {@code Class} object of the enum type from which
 *      to return a constant
 * @param name the name of the constant to return
 * @return the enum constant of the specified enum type with the
 *      specified name
 * @throws IllegalArgumentException if the specified enum type has
 *         no constant with the specified name, or the specified
 *         class object does not represent an enum type
 * @throws NullPointerException if {@code enumType} or {@code name}
 *         is null
 * @since 1.5
 */
public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                            String name) {
    T result = enumType.enumConstantDirectory().get(name);
    if (result != null)
        return result;
    if (name == null)
        throw new NullPointerException("Name is null");
    throw new IllegalArgumentException(
        "No enum constant " + enumType.getCanonicalName() + "." + name);
}
```

根据注释我们可以知道：

- 该函数的功能时根据枚举名称和枚举类型找到对应的枚举常量。
- 所有的枚举类型有一个隐式的函数 `public static T valueOf(String)` 用来根据枚举名称来获取枚举常量。
- 如果想获取当前枚举的所有枚举常量可以通过调用隐式的 `public static T[] values()` 函数来实现。

另外一个就是上面提到的 `public static com.imooc.basic.learn_enum.CoinEnum[] values();` 函数。

我们回到上面反汇编的代码，偏移为 58 到 96 的指令转为 Java **代码效果**和下面很类似：

```java
  private static CoinEnum[] $VALUES;
  static {
      $VALUES = new CoinEnum[4];
      $VALUES[0] = PENNY;
      $VALUES[1] = NICKEL;
      $VALUES[2] = DIME;
      $VALUES[3] = QUARTER;
  }
```

根据第 1 处代码

```java
 // 第 1 处代码
  public static com.imooc.basic.learn_enum.CoinEnum[] values();
    Code:
       0: getstatic     #1                  // Field $VALUES:[Lcom/imooc/basic/learn_enum/CoinEnum;
       3: invokevirtual #2                  // Method "[Lcom/imooc/basic/learn_enum/CoinEnum;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[Lcom/imooc/basic/learn_enum/CoinEnum;"
       9: areturn
```

我们可以大致还原成下面的代码：

```java
 public static CoinEnum[] values() {
     return $VALUES.clone();
 }
```

因此整体的逻辑就很清楚了。

结合前面拷贝章节讲到的内容，接下来大家思考下一个新问题：**为什么返回克隆对象而不是属性里的枚举数组呢？**

其实这样设计的主要原因是：避免枚举数组在外部进行修改，影响到下一次调用：`CoinEnum.values()` 的结果。如：

```java
 @Test
 public void testValues(){
      CoinEnum[] values1 = CoinEnum.values();
      values1[0] = CoinEnum.QUARTER;

      CoinEnum[] values2 = CoinEnum.values();
      Assert.assertEquals(values2[0],CoinEnum.PENNY);
 }
```

通过上面代码片段可以看出：对通过 clone 函数构造的新的数组对象（values1）的某个元素重新赋值并不会影响到原数组。

因此再次调用 `CoinEnum.values()` 仍然会返回基于原始枚举数组创建的新的拷贝对象（values2）。



##### 2.4 源码大法

通过官方文档和反汇编，我们知道：枚举类都是 `java.lang.Enum` 的子类型。正因如此，我们可以通过查看 `Enum` 类的源码来学习枚举的一些知识。

我们通过 IDEA 自带的 Diagrams -> Show Diagrams -> Java Class Diagram 可以看到 Enum 类的继承关系，以及属性和函数等信息。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

可以看到实现了 `Comparable` 和 `Serializable` 接口。

**那么为什么要实现这两个接口？**

- 实现 `Comparable` 接口很好理解，是为了排序。
- 实现 `Serializable` 接口是为了序列化。

前面序列化的小节中讲到：“一个类实现序列化接口，那么其子类也具备序列化的能力。”

从这里大家就会明白，正是因为其父类 `Enum` 实现了序列化接口，我们的枚举类没有显式实现序列化接口，使用 Java 原生序列化也并不会报错。

其中 `Enum` 类有两个属性 **：

`name` 表示枚举的名称。

`ordinal` 表示枚举的顺序，其主要用在 `java.util.EnumSet` 和 `java.util.EnumMap` 这两种基于枚举的数据结构中。

感兴趣的同学可以继续研究这两个数据结构的用法。

接下来我带大家重点看两个函数的源码： `java.lang.Enum#clone` 函数和 `java.lang.Enum#compareTo` 函数。

我们查看 `Enum` 类的 `clone` 函数：

```java
/**
 * Throws CloneNotSupportedException.  This guarantees that enums
 * are never cloned, which is necessary to preserve their "singleton"
 * status.
 *
 * @return (never returns)
 */
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

通过注释和源码我们可以明确地学习到，枚举类不支持 `clone` , 如果调用会报 `CloneNotSupportedException` 异常。

**目的是为了保证枚举不能被克隆，维持单例的状态**。

我们知道即使将构造方法设置为私有，也可以通过反射机制 `setAccessible` 为 `true` 后调用。普通的类可以通过 `java.lang.reflect.Constructor#newInstance` 来构造实例，这样就破坏了单例。

然而在该函数源码中对枚举类型会作判断并报 `IllegalArgumentException`。

```java
public T newInstance(Object ... initargs)
    throws InstantiationException, IllegalAccessException,
           IllegalArgumentException, InvocationTargetException
{
    // 省略..
    if ((clazz.getModifiers() & Modifier.ENUM) != 0)
        throw new IllegalArgumentException("Cannot reflectively create enum objects");
 
     // 省略..
    return inst;
}
```

这样就防止了通过反射来构造枚举实例的可能性。

接下来我们看 `compareTo` 函数源码：

```java
/**
 * Compares this enum with the specified object for order.  Returns a
 * negative integer, zero, or a positive integer as this object is less
 * than, equal to, or greater than the specified object.
 *
 * Enum constants are only comparable to other enum constants of the
 * same enum type.  The natural order implemented by this
 * method is the order in which the constants are declared.
 */
public final int compareTo(E o) {
    Enum<?> other = (Enum<?>)o;
    Enum<E> self = this;
    if (self.getClass() != other.getClass() && // optimization
        self.getDeclaringClass() != other.getDeclaringClass())
        throw new ClassCastException();
    return self.ordinal - other.ordinal;
}
```

根据注释和源码，我们可以看到：其排序的依据是 枚举常量在枚举类的声明顺序。



##### 2.5 断点大法

那么我们想想为啥《手册》中会有下面的这个规定呢？

> 【强制】二方库里可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚举类型或者包含枚举类型的 POJO 对象。
>
> 注：
>
> 二方是指公司内部的其他部门；
>
> 二方库是指公司内部发布到中央仓库，可供公司内部其他应用依赖的库（jar 包）。

我们写一个测试函数来研究这个问题：

```java
@Test
public void serialTest() {
    CoinEnum[] values = CoinEnum.values();
    // 序列化
    byte[] serialize = SerializationUtils.serialize(values);

    log.info("序列化后的字符：{}",new String(serialize));
    // 反序列化
    CoinEnum[] values2 = SerializationUtils.deserialize(serialize);

    Assert.assertTrue(Objects.deepEquals(values, values2));
}
```

我们在 `java.lang.Enum#valueOf` 函数第一行打断点。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

大家一定要自己尝试双击左下角的调用栈部分，查看从顶层调用

`org.apache.commons.lang3.SerializationUtils#deserialize(byte[])` 到

`java.lang.Enum#valueOf` 的整个调用过程。大家还可以通过表达式来查看参数的各种属性。

可以看到枚举的反序列化是通过调用 `java.lang.Enum#valueOf` 来实现的 **。

另外我们可以查看序列化后的字节流的字符表示形式：

序列化后的字符：

> ��ur&[Lcom.imooc.basic.learn_enum.CoinEnum;ċ���>��xpr#com.imooc.basic.learn_enum.CoinEnumxrjava.lang.EnumxptPENNYqtNICKELqtDIMEq~tQUARTER

大致可以看出，序列化后的数据中主要包含**枚举的类型**和**枚举名称**。

我们了解了枚举的序列化和反序列化的原理后我们再思考：**为什么接口返回值不允许使用枚举类型或者包含枚举类型的 POJO 对象？**

上面讲到反序列化枚举类会调用 `java.lang.Enum#valueOf` ：

```java
/**
     * Returns the enum constant of the specified enum type with the
     * specified name.  The name must match exactly an identifier used
     * to declare an enum constant in this type.  (Extraneous whitespace
     * characters are not permitted.)
     *
     * <p>Note that for a particular enum type {@code T}, the
     * implicitly declared {@code public static T valueOf(String)}
     * method on that enum may be used instead of this method to map
     * from a name to the corresponding enum constant.  All the
     * constants of an enum type can be obtained by calling the
     * implicit {@code public static T[] values()} method of that
     * type.
     *
     * @param <T> The enum type whose constant is to be returned
     * @param enumType the {@code Class} object of the enum type from which
     *      to return a constant
     * @param name the name of the constant to return
     * @return the enum constant of the specified enum type with the
     *      specified name
     * @throws IllegalArgumentException if the specified enum type has
     *         no constant with the specified name, or the specified
     *         class object does not represent an enum type
     * @throws NullPointerException if {@code enumType} or {@code name}
     *         is null
     * @since 1.5
     */
    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }
```

大家可以设想一下，如果将枚举当做 RPC 接口的返回值或者返回值对象的属性。如果己方接口新增枚举常量，而二方（公司的其他部门）没有及时升级 JAR 包，会出现什么情况？

此时，如果己方调用此接口时传入新的枚举常量，进行序列化。

反序列化时会调用到 `java.lang.Enum#valueOf` 函数， 此时参数 `name` 值为新的枚举名称。

```java
T result = enumType.enumConstantDirectory().get(name);
```

此时 `result = null` ，从源码可以看出，将会抛出 `IllegalArgumentException` 。

通过查看该函数顶部的 `@throws IllegalArgumentException` 注释，我们也可以得知：

> 如果枚举类没有该常量，或者该反序列化的类对象并不是枚举类型则会抛出该异常。

因此，二方的枚举类添加新的常量后，如果使用方没有及时更新 JAR 包，使用 Java 反序列化时可能会抛出 `IllegalArgumentException` 。

除了 Java 序列化、反序列化外，其他的序列化框架对于枚举类处理也容易出现各种错误，因此请严格遵守这一条。

大家可以通过为 `CoinEnum` 枚举类新增一个枚举常量，并将新增的枚举常量通过 Java 序列化到文件中，然后在源码中注释掉新增的枚举常量，再反序列化，来复现这个 BUG。

**有没有好的解决办法？**

最常见的做法就是返回枚举的数值，并在返回的包中给出枚举类，在枚举类中提供通过根据值去获取枚举常量的方法（具体做法见下文）。

并通过使用 `@see` 或 `{@link}` 在该返回的枚举的数值注释中给出指向枚举类的快捷方式，如：

```java
/**
 * 硬币值，对应的枚举参见{@link CoinEnum}
 */
private Integer coinValue;
```



#### 3. 根据值获取枚举常量的用法

偶尔会遇到有些团队实现通过枚举中的值获取枚举常量时，居然用 switch ，非常让人吃惊。

如上面的 `CoinEnum` 的根据值获取枚举的函数，有些人会这么写：

```java
public static CoinEnum getEnum(int value) {
        switch (value) {
            case 1:
                return PENNY;
            case 5:
                return NICKEL;
            case 10:
                return DIME;
            case 25:
                return QUARTER;
            default:
                return null;
        }
    }
```

这样做不符合设计模式的六大原则之一的 “开闭原则”，因为如果删除、新增一个枚举常量等，也需要修改该函数。

> 开闭原则：对拓展开放，对修改关闭。

另外如果枚举常量较多，很容易映射错误，后期很难维护。

可以利用前面讲到的枚举的 values 函数实现该功能，参考写法如下：

```java
public static CoinEnum getEnum(int value) {
    for (CoinEnum coinEnum : CoinEnum.values()) {
        if (coinEnum.value == value) {
            return coinEnum;
        }
    }
    return null;
}
```

使用上面的写法，如果后面需要对枚举常量进行修改，该函数不需要改动，显然比之前好了很多。

实际工作中这种写法也很常见。

**那么还有改进空间吗？**

这种写法虽然挺不错，但是每次获取枚举对象都要遍历一次枚举数组，时间复杂度是 O (n)。

降低时间复杂度该怎么做？一个常见的思路就是**空间换时间**。

因此我们可以事先通过 Map 将映射关系存起来，使用时直接从 Map 中获取，参考代码如下：

```java
@Getter
public enum CoinEnum {
    PENNY(1), NICKEL(5), DIME(10), QUARTER(25)/*,NEWONE(50)*/;

    CoinEnum(int value) {
        this.value = value;
    }

    private final int value;

    public int value() {
        return value;
    }

    private static final Map<Integer, CoinEnum> cache = new HashMap<>();

    static {
        for (CoinEnum coinEnum : CoinEnum.values()) {
            cache.put(coinEnum.getValue(), coinEnum);
        }
    }

    public static CoinEnum getEnum(int value) {
        return cache.getOrDefault(value, null);
    }
}
```

通过上面的优化，使用时时间复杂度为 O (1)，性能有所提升。

**那么还有改进的空间吗？**

上面的代码还存在以下几个问题：

- 每个枚举类中都需要编写类似的代码，很繁琐。
- 引入提供上述工具的很多枚举类，如果仅使用枚举常量，也会触发静态代码块的执行。

可不可以不修改枚举就能具备这种功能？是不是可以抽取公共部分代码封装成工具类？

我们来试一试。

首先大家可以想想，如果我们要将这部分封装成工具函数，需要哪些参数？

显然需要枚举的类型，还需要知道枚举中哪个属性作为缓存的 key，还需要传入匹配的参数。

因此可以编写如下工具类封装获取枚举对象的方法：

```java
mport java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;

public class EnumUtils {

    private static final Map<Object, Object> key2EnumMap = new ConcurrentHashMap<>();

    private static final Set<Class> enumSet = ConcurrentHashMap.newKeySet();


    /**
     * 带缓存的获取枚举值方式
     *
     * @param enumType    枚举类型
     * @param keyFunction 根据枚举类型获取key的函数
     * @param key         带匹配的Key
     * @param <T>         枚举泛型
     * @return 枚举类型
     */
    public static <T extends java.lang.Enum<T>> Optional<T> getEnumWithCache(Class<T> enumType, Function<T, Object> keyFunction, Object key) {

        if (!enumSet.contains(enumType)) {
            // 不同的枚举类型相互不影响
            synchronized (enumType) {
                if (!enumSet.contains(enumType)) {
                    // 添加枚举
                    enumSet.add(enumType);
                    // 缓存枚举键值对
                    for (T enumThis : enumType.getEnumConstants()) {
                        // 避免重复
                        String mapKey = getKey(enumType, keyFunction.apply(enumThis));

                        key2EnumMap.put(mapKey, enumThis);
                    }

                }
            }
        }
         return Optional.ofNullable((T) key2EnumMap.get(getKey(enumType, key)));
    }

    /**
     * 获取key
     * 注：带上枚举路径避免不同枚举的Key 重复
     */
    public static <T extends java.lang.Enum<T>> String getKey(Class<T> enumType, Object key) {

        return enumType.getName().concat(key.toString());
    }

    /**
     * 不带缓存的获取枚举值方式
     *
     * @param enumType    枚举类型
     * @param keyFunction 根据枚举类型获取key的函数
     * @param key         带匹配的Key
     * @param <T>         枚举泛型
     * @return 枚举类型
     */
    public static <T extends java.lang.Enum<T>> Optional<T> getEnum(Class<T> enumType, Function<T, Object> keyFunction, Object key) {
        for (T enumThis : enumType.getEnumConstants()) {
            if (keyFunction.apply(enumThis).equals(key)) {
                return Optional.of(enumThis);
            }
        }
        return Optional.empty();
    }
}
```

> 注：上述的几种写法，仅适合枚举常量和对应的属性一对一的情况，其他场景可能要换一种写法。
>
> 另外建议大家再思考下此方案还有没有优化的空间？是否还有其他优雅解决方案？

使用也非常简单:

```java
@Test
public void test() {
    int key = 5;

    CoinEnum targetEnum = CoinEnum.NICKEL;

    CoinEnum anEnum = CoinEnum.getEnum(key);
    Assert.assertEquals(targetEnum, anEnum);

    // 使用缓存
    Optional<CoinEnum> enumWithCache = EnumUtils.getEnumWithCache(CoinEnum.class, CoinEnum::getValue, key);
    Assert.assertTrue(enumWithCache.isPresent());
    Assert.assertEquals(targetEnum, enumWithCache.get());

    // 不使用缓存（遍历）
    Optional<CoinEnum> enumResult = EnumUtils.getEnum(CoinEnum.class, CoinEnum::getValue, key);
    Assert.assertTrue(enumResult.isPresent());
    Assert.assertEquals(targetEnum, enumResult.get());
}
```

使用上面封装的工具类，不仅能够满足功能要求，还能实现了代码的复用，同时也做到了性能的优化。

通过上面的讲解，希望大家明白 “尽信书不如无书” 的道理，不要因为看到某个博客、某本书给出一个不错的写法就认为是标准答案，要有自己的思考，要有一定的代码优化意识。



#### 4. 枚举的高级用法



##### 4.1 实现计算

从官方文档中我们可以看到，枚举常量可以带类方法：

```java
enum Operation {
    PLUS {
        double eval(double x, double y) { return x + y; }
    },
    MINUS {
        double eval(double x, double y) { return x - y; }
    },
    TIMES {
        double eval(double x, double y) { return x * y; }
    },
    DIVIDED_BY {
        double eval(double x, double y) { return x / y; }
    };

    // Each constant supports an arithmetic operation
    abstract double eval(double x, double y);

    public static void main(String args[]) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        for (Operation op : Operation.values())
            System.out.println(x + " " + op + " " + y +
                               " = " + op.eval(x, y));
    }
}
```

可以在枚举类中定义抽象方法，在枚举常量中实现该方法来提供计算等功能.

JDK 源码中常见的枚举类： `java.util.concurrent.TimeUnit` 类就有类似的用法。

这种策略枚举方式也是替代 if - else if - else 的一种解决方案。



##### 4.2 实现状态机

假设业务开发中需要实现状态流转的功能。

活动有：申报 -> 批准 -> 报名 -> 开始 -> 结束几种状态，依次流转。

我们可以通过下面的代码实现：

```java
public enum ActivityStatesEnum {
    /**
     * 活动状态
     * 申报-> 批准-> 报名 -> 开始 -> 结束
     */
    DEACLARE(1) {
        @Override
        ActivityStatesEnum nextState() {
            return APPROVE;
        }
    },
    APPROVE(2) {
        @Override
        ActivityStatesEnum nextState() {
            return ENROLL;
        }
    },
    ENROLL(3) {
        @Override
        ActivityStatesEnum nextState() {
            return START;
        }
    },
    START(4) {
        @Override
        ActivityStatesEnum nextState() {
            return END;
        }
    },
    END(5) {
        @Override
        ActivityStatesEnum nextState() {
            return this;
        }
    };

    private int status;

    abstract ActivityStatesEnum nextState();

    ActivityStatesEnum(int status) {
        this.status = status;
    }
  
  public ActivityStatesEnum getEnum(int status) {
        for (ActivityStatesEnum statesEnum : ActivityStatesEnum.values()) {
            if (statesEnum.status == status) {
                return statesEnum;
            }
        }
        return null;
    }
}
```

这样做的好处是可以通过 `getEnum` 函数获取枚举，直接通过 `nextState` 来获取下一个状态，更容易封装状态流转的函数，不需要每个状态都通过 `if` 判断再指定下一个状态，也降低出错的概率。



##### 4.3 灵活的特性组合

fastjson 的 `com.alibaba.fastjson.parser.Feature` 类，灵活使用 `java.lang.Enum#ordinal` 和位运算实现了灵活的特性组合。

源码如下：

```java
 public enum Feature {

    AutoCloseSource,
   
   // 省略了一部分代码
   
   Feature(){
        mask = (1 << ordinal());
    }

    public final int mask;

    public final int getMask() {
        return mask;
    }

    public static boolean isEnabled(int features, Feature feature) {
        return (features & feature.mask) != 0;
    }

    public static int config(int features, Feature feature, boolean state) {
        if (state) {
            features |= feature.mask;
        } else {
            features &= ~feature.mask;
        }

        return features;
    }
    
    public static int of(Feature[] features) {
        if (features == null) {
            return 0;
        }
        
        int value = 0;
        
        for (Feature feature: features) {
            value |= feature.mask;
        }
        
        return value;
    }
 }
```

我们知道 `java.lang.Enum#ordinal` 表示枚举序号。因此可以通过将 1 左移枚举序号个位置，构造各种特性的掩码。

各种特性的掩码可以任意组合，来表示不同的特征组合，也可以根据特性值反向解析出这些特性组合。



#### 5. 总结

本节使用的学习方法有，思考技术的初衷，官方文档，读源码和反汇编。

主要要点如下：

1. 枚举一般表示相同类型的常量。
2. 枚举隐式继承自 `Enum` ，实现了 `Comparable` 和 `Serializable` 接口。
3. `java.util.EnumSet` 和 `java.util.EnumMap` 是两种关于 `Enum` 的数据结构。
4. 枚举类可以使用其 `ordinal` 属性，通过定义抽象函数、实现接口等方式实现高级用法。

更多枚举进阶知识可参考《Effective Java》 第 6 章 枚举和注解。

下一节将讲述 `ArrayList` 类的 `subList` 函数和 `Arrays` 类的 `asList` 函数。



#### 课后题

1、通过前几节介绍的 `codota` 来学习两种和 `Enum` 相关的数据结构 ：`java.util.EnumSet` 和 `java.util.EnumMap` 的用法。

2、请为 `CoinEnum` 枚举类新增一个枚举常量，并将新增的枚举常量通过 Java 序列化到文件中，然后注释掉源码中新增的枚举常量，再反序列化，观察效果。



### ArrayList的subList和Arrays的asList学习



#### 1. 前言

《手册》 第 11-12 页 对 `ArrayList` 的 `subList` 和 `Arrays.asList()` 进行了如下描述 [1](https://www.imooc.com/read/55/article/1148#fn1)：

> 【强制】ArrayList 的 subList 结果不可强转成 ArrayList，否则会抛出 ClassCastException 异 常，即 java.util.RandomAccessSubList cannot be cast to java.util.ArrayList。
>
> 【强制】在 SubList 场景中，高度注意对原集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生 ConcurrentModificationException 异常。
>
> 【强制】使用工具类 Arrays.asList () 把数组转换成集合时，不能使用其修改集合相关的方法，它的 add/remove/clear 方法会抛出 UnsupportedOperationException 异常。

那么我们思考下面几个问题：

- 《手册》为什么要这么规定？
- 这对我们编码又有什么启发呢？

这些都是本节重点解答的问题。



#### 2. 问题分析

通过前面章节的学习，相信很多人已经对通过使用类图、阅读源码和源码的注释等来学习方法已经轻车熟路了。

下面我们根据本节话题继续实战。



##### 2.1 ArrayList 的 subList 分析



###### 2.1.1 类图法

通过 IDEA 的提供的类图工具，我们可以查看该类的继承体系。

具体步骤：在 `SubList` 类中 右键，选择 “Diagrams” -> “Show Diagram” 。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

可以看到 `SubList` 和 `ArrayList` 的继承体系非常类似，都实现了 `RandomAccess` 接口 继承自 `AbstarctList。`

`SubList` 和 `ArrayList` 并没有继承关系，因此 “ `ArrayList` 的 `SubList` 并不能强转为 `ArrayList` 。

通过类图我们对 `SubList` 有了一个整体的了解，这将为我们进步学习打下很好的基础。



###### 2.2.2 DEMO 和调试大法

如果想**学习某个特性，最好的方法之一就是写一个小段 DEMO 来观察分析**。

因此我们下面，写一个简单的测试代码片段来验证转换异常问题：

```java
@Test(expected = ClassCastException.class)
public void testClassCast() {
    List<Integer> integerList = new ArrayList<>();
    integerList.add(0);
    integerList.add(1);
    integerList.add(2);
    List<Integer> subList = integerList.subList(0, 1);
    
    // 强转
    ArrayList<Integer> cast = (ArrayList<Integer>) subList;
}
```

我们还可以使用调试的表达式功能来验证我们的想法。

在调试界面的 “Variables” 窗口选择想研究的对象，如 `subList` ，然后右键选择 “Evaluate Expression”，输入想查执行的表达式，查看结果：
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

从上面的表达式的结果也可以清晰地看出，`subList` 并不是 `ArrayList` 类型的实例。

我们写一个代码片段来验证功能：

```java
@Test
public void testSubList() {
        List<String> stringList = new ArrayList<>();
        stringList.add("赵");
        stringList.add("钱");
        stringList.add("孙");
        stringList.add("李");
        stringList.add("周");
        stringList.add("吴");
        stringList.add("郑");
        stringList.add("王");

        List<String> subList = stringList.subList(2, 4);
        System.out.println("子列表：" + subList.toString());
        System.out.println("子列表长度：" + subList.size());

        subList.set(1, "慕容");
        System.out.println("子列表：" + subList.toString());
        System.out.println("原始列表：" + stringList.toString());
 }
```

输出结果为：

> 子列表：[孙，李]
> 子列表长度：2
> 子列表：[孙，慕容]
> 原始列表：[赵，钱，孙，慕容，周，吴，郑，王]

可以观察到，对子列表的修改最终对原始列表产生了影响。

那么为啥修改子序列的索引为 1 的值影响的是原始列表的第 4 个元素呢？后面将进行分析和解读。



###### 2.1.3 源码分析

`java.util.ArrayList#subList` 源码：

```java
/**
 * Returns a view of the portion of this list between the specified
 * {@code fromIndex}, inclusive, and {@code toIndex}, exclusive.  (If
 * {@code fromIndex} and {@code toIndex} are equal, the returned list is
 * empty.)  The returned list is backed by this list, so non-structural
 * changes in the returned list are reflected in this list, and vice-versa.
 * The returned list supports all of the optional list operations.
 *
 * <p>This method eliminates the need for explicit range operations (of
 * the sort that commonly exist for arrays).  Any operation that expects
 * a list can be used as a range operation by passing a subList view
 * instead of a whole list.  For example, the following idiom
 * removes a range of elements from a list:
 * <pre>
 *      list.subList(from, to).clear();
 * </pre>
 * Similar idioms may be constructed for {@link #indexOf(Object)} and
 * {@link #lastIndexOf(Object)}, and all of the algorithms in the
 * {@link Collections} class can be applied to a subList.
 *
 * <p>The semantics of the list returned by this method become undefined if
 * the backing list (i.e., this list) is <i>structurally modified</i> in
 * any way other than via the returned list.  (Structural modifications are
 * those that change the size of this list, or otherwise perturb it in such
 * a fashion that iterations in progress may yield incorrect results.)
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 * @throws IllegalArgumentException {@inheritDoc}
 */
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}
```

通过源码可以看到该方法主要有两个核心逻辑：一个是检查索引的范围，一个是构造子列表对象。

通注释我们可以学到核心知识点：

> 该方法返回本列表中 fromIndex （包含）和 toIndex （不包含）之间的**元素视图**。如果两个索引相等会返回一个空列表。
>
> 如果需要对 list 的某个范围的元素进行操作，可以用 subList，如：
>
> list.subList(from, to).clear();
>
> 任何对子列表的操作最终都会反映到原列表中。

我们查看函数 `java.util.ArrayList.SubList#set` 源码：

```java
public E set(int index, E e) {
    rangeCheck(index);
    checkForComodification();
    E oldValue = ArrayList.this.elementData(offset + index);
    ArrayList.this.elementData[offset + index] = e;
    return oldValue;
}
```

可以看到替换值的时候，获取索引是通过 `offset + index` 计算得来的。

这里的 `java.util.ArrayList#elementData` 即为原始列表存储元素的数组。

```java
SubList(AbstractList<E> parent,
        int offset, int fromIndex, int toIndex) {
    this.parent = parent;
    this.parentOffset = fromIndex;
    this.offset = offset + fromIndex;
    this.size = toIndex - fromIndex;
    this.modCount = ArrayList.this.modCount; // 注意：此处复制了 ArrayList的 modCount
}
```

通过子列表的构造函数我们知道，这里的偏移量 ( `offset` ) 的值为 `fromIndex` 参数。

因此上小节提到的：** 为啥子序列的索引为 1 的值影响的是原始列表的第 4 个元素呢？** 的问题就不言自明了。

另外在 `SubList` 的构造函数中，会将 `ArrayList` 的 `modCount` 赋值给 `SubList` 的 `modCount` 。

我们再回到规约中规定：

> 【强制】在 subList 场景中，高度注意对原集合元素的增加或删除，均会导致子列表的遍历、增加、删除产生 ConcurrentModificationException 异常。

我们看 `java.util.ArrayList#add(E)` 的源码：

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return <tt>true</tt> (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

可以发现新增元素和删除元素，都会对 `modCount` 进行修改。

我们再看 `SubList` 的 核心的函数，如 `java.util.ArrayList.SubList#get` 和 `java.util.ArrayList.SubList#size` ：

```java
public E get(int index) {
    rangeCheck(index);
    checkForComodification();
    return ArrayList.this.elementData(offset + index);
}

public int size() {
    checkForComodification();
    return this.size;
}
```

都会进行修改检查：

```
java.util.ArrayList.SubList#checkForComodification
private void checkForComodification() {
    if (ArrayList.this.modCount != this.modCount)
        throw new ConcurrentModificationException();
}
```

而从上面的 `SubList` 的构造函数我们可以看到，`SubList` 复制了 ArrayList 的 modCount，因此对原函数的新增或删除都会导致 `ArrayList` 的 `modCount` 的变化。而子列表的遍历、增加、删除时又会检查创建 `SubList` 时的 modCount 是否一致，显然此时两者会不一致，导致抛出 `ConcurrentModificationException` (并发修改异常)。

至此上面约定的原因我们也非常明了了。



##### 2.2 Arrays.asList () 分析



###### 2.2.1 类图法

和前面一样，查看类图来了解 `Arrays.asList()` 的返回类型。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

发现该 `java.util.Arrays.ArrayList` (右侧) 和 `java.util.ArrayList` （左侧），的继承体系非常相似，继承自 `java.util.AbstractList` 。

我们打开左上角的 “Method” 功能，对比两者的主要函数的异同：
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

我们可以清楚地发现， `java.util.Arrays.ArrayList` (右侧) 并没有像左侧一样 重写 `add` 、 `remove` 函数。



###### 2.2.2 源码大法

接下来我们分析 `Arrays.asList()` 的源码：

```java
/**
 * Returns a fixed-size list backed by the specified array.  (Changes to
 * the returned list "write through" to the array.)  This method acts
 * as bridge between array-based and collection-based APIs, in
 * combination with {@link Collection#toArray}.  The returned list is
 * serializable and implements {@link RandomAccess}.
 *
 * <p>This method also provides a convenient way to create a fixed-size
 * list initialized to contain several elements:
 * <pre>
 *     List&lt;String&gt; stooges = Arrays.asList("Larry", "Moe", "Curly");
 * </pre>
 *
 * @param <T> the class of the objects in the array
 * @param a the array by which the list will be backed
 * @return a list view of the specified array
 */
@SafeVarargs
@SuppressWarnings("varargs")
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```

通过注释我们可以得到下面的要点：

> 返回基于特定数组的**定长列表**。
>
> 该方法扮演数组到集合的桥梁。
>
> 该方法也提供了包含多个元素的定长列表的方法：
>
> List stooges = Arrays.asList(“Larry”, “Moe”, “Curly”);

可看出此方法的功能是为了返回**定长的列表**。

这里的” 定长列表 “的描述非常重要，这也就解释了为什么不支持增加和删除元素的原因。

结合前面的类图，我们去查看 `AbstactList` 的 `add` 和 `remove` 相关函数：

```
java.util.AbstractList#add(int, E)
public void add(int index, E element) {
    throw new UnsupportedOperationException();
}
java.util.AbstractList#remove
public E remove(int index) {
    throw new UnsupportedOperationException();
}
```

可知如果子类不重写这两个函数，就会抛出 `UnsupportedOperationException`（不支持的操作异常）。

我们再看看 `java.util.AbstractList#clear` 的源码：

```java
/**
 * Removes all of the elements from this list (optional operation).
 * The list will be empty after this call returns.
 *
 * <p>This implementation calls {@code removeRange(0, size())}.
 *
 * <p>Note that this implementation throws an
 * {@code UnsupportedOperationException} unless {@code remove(int
 * index)} or {@code removeRange(int fromIndex, int toIndex)} is
 * overridden.
 *
 * @throws UnsupportedOperationException if the {@code clear} operation
 *         is not supported by this list
 */
public void clear() {
    removeRange(0, size());
}
```

通过注释可知 如果没有重写 `remove(int index)` 或 `removeRange(int fromIndex, int toIndex)` 同样也会抛出 `UnsupportedOperationException` 。



#### 3. 学习的启发

在 Java 的学习过程中，大多数人都是通过看视频，读博客，搜索引擎搜索，买书等来学习知识。

但是很多资料都是告诉你结论，但这样容易浮于表面，知其然而不知其所以然。而源码、官方文档等才是权威的知识。

希望从现在开始学习和开发中能够偶尔到感兴趣的类中查看源码，这样学的更快，更扎实。通过进入源码中自主研究，这样印象更加深刻，掌握的程度更深。

我们同样发现学习的手段并非只有一种，往往多种研究方式结合起来效果最好。



#### 4. 总结

本文通过类图分析、源码分析以及 DEMO 和调试的方式对 `ArrayList` 的 `SubList` 问题和 `Arrays` 的 `asList` 进行分析。并根据分析阐述了对我们学习的启发。

本节的要点：

1. `ArrayList` 内部类 `SubList` 和 `ArrayList` 没有继承关系，因此无法将其强转为 `ArrayList` 。
2. `ArrayList` 的 `SubList` 构造时传入 `ArrayList` 的 `modCount`，因此对原列表的修改将会导致子列表的遍历、增加、删除产生 ConcurrentModificationException 异常。
3. `Arrays.asList()` 函数是提供通过数组构造定长集合的功能，该函数提供数组到集合的桥梁。

下一节我们将讲述添加注释的正确姿势。



#### 5. 课后练习

《手册》第 11 页 集合处理章节有这么一条规定：

> 【强制】不要在 foreach 循环里进行元素的 remove/add 操作。remove 元素请使用 Iterator 方式，如果并发操作，需要对 Iterator 对象加锁。

那么问题来了，为什么 “不要在 foreach 循环里进行元素的 remove/add 操作。remove 元素请使用 Iterator 方式”？

请大家结合前面和本小节所学的内容自己实际动手研究一下。



### 添加注释的正确姿



#### 1. 前言

《手册》 21 页，第八节 注释规约部分对注释规范的要点给出了比较全面的指导 [1](https://www.imooc.com/read/55/article/1149#fn1)。

> 【强制】所有类都必须添加创建者和日期。
>
> 【强制】所有的枚举类型字段都必须有注释，说明每个数据项的用途。
>
> 【推荐】代码修改的同时，注释也要进行相应的修改，尤其是参数、返回值、异常、核心逻辑等修改。
>
> 【参考】特殊标记，请注明标记人与标记时间。

我们要思考以下几个问题：

- 你平时写注释吗？
- 你知道注释的目的是什么？
- 有哪些好的注释范例？
- 为什么会有这些规定？
- 还有哪些好的规约？

本节将为你解答上述疑问。



#### 2 注释的目的

注释的目的是：**辅助读代码的人员更快速的理解代码**。

因此我们写注释的时候不管使用何种规约和技巧都要围绕这个目的展开。

这就要求编写注释时，要能够**准确描述函数的功能，核心逻辑，潜在风险，注意事项等**。

如果注释写地好，即使过了很久自己可以通过注释快速理解代码，也可以帮助团队其他合作的成员快速理解自己的代码，快速找到相关文档，也将方便未来接手自己工作的开发人员。这也是一个优秀程序员专业性的一种体现。



#### 3 常见的注释类型和写法



##### 3.1 常规注释

常规注释主要指普通的注释，比如每个接口几乎都会有的：接口的功能，接口的参数以及含义，接口异常和出现异常的原因，接口的返回值。

首先我们从 JDK 代码注释中寻找灵感。

我们可以参考 `ThreadPoolExecutor#ThreadPoolExecutor` 构造函数的注释：

```java
/**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters.
 *
 * @param corePoolSize the number of threads to keep in the pool, even
 *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
 * @param maximumPoolSize the maximum number of threads to allow in the
 *        pool
 * @param keepAliveTime when the number of threads is greater than
 *        the core, this is the maximum time that excess idle threads
 *        will wait for new tasks before terminating.
 * @param unit the time unit for the {@code keepAliveTime} argument
 * @param workQueue the queue to use for holding tasks before they are
 *        executed.  This queue will hold only the {@code Runnable}
 *        tasks submitted by the {@code execute} method.
 * @param threadFactory the factory to use when the executor
 *        creates a new thread
 * @param handler the handler to use when execution is blocked
 *        because the thread bounds and queue capacities are reached
 * @throws IllegalArgumentException if one of the following holds:<br>
 *         {@code corePoolSize < 0}<br>
 *         {@code keepAliveTime < 0}<br>
 *         {@code maximumPoolSize <= 0}<br>
 *         {@code maximumPoolSize < corePoolSize}
 * @throws NullPointerException if {@code workQueue}
 *         or {@code threadFactory} or {@code handler} is null
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

可以看到该注释先给出了该构造函数的**功能说明**，然后对**每个参数的含义**进行解读，然后给出了**抛出的异常以及抛出异常对应的具体原因**。

正是 JDK 的注释非常专业和详细，才为我们学习源码提供了便利。试想如果没有注释，我们学习和理解源码的速度会不会更慢呢？



##### 3.2 工具函数注释

工具类的注释主要包含：函数的功能，函数的使用范例，函数的参数和返回值描述，该函数出现的起始版本等。

我们选取 commons-lang3 的 `StringUtils` 类的 `StringUtils#isAnyEmpty` 函数的源码来学习工具函数的注释。

```java
/**
 * <p>Checks if any of the CharSequences are empty ("") or null.</p>
 *
 * <pre>
 * StringUtils.isAnyEmpty((String) null)    = true
 * StringUtils.isAnyEmpty((String[]) null)  = false
 * StringUtils.isAnyEmpty(null, "foo")      = true
 * StringUtils.isAnyEmpty("", "bar")        = true
 * StringUtils.isAnyEmpty("bob", "")        = true
 * StringUtils.isAnyEmpty("  bob  ", null)  = true
 * StringUtils.isAnyEmpty(" ", "bar")       = false
 * StringUtils.isAnyEmpty("foo", "bar")     = false
 * StringUtils.isAnyEmpty(new String[]{})   = false
 * StringUtils.isAnyEmpty(new String[]{""}) = true
 * </pre>
 *
 * @param css  the CharSequences to check, may be null or empty
 * @return {@code true} if any of the CharSequences are empty or null
 * @since 3.2
 */
public static boolean isAnyEmpty(final CharSequence... css) {
  if (ArrayUtils.isEmpty(css)) {
    return false;
  }
  for (final CharSequence cs : css){
    if (isEmpty(cs)) {
      return true;
    }
  }
  return false;
}
```

除了前面提到的功能描述，参数和返回值描述外，该注释部分还给出了**常见的使用范例和执行结果**，能够帮助读者快速理解函数的用法。

强烈建议我们在编写工具类时参考这种写法，方便使用者的同时也体现了我们的专业水准。



##### 3.3 废弃代码的注释

正如专栏的 ” 过期类、属性和接口的正确处理方式 “ 小节所讲的：废弃的接口要给出废弃的原因和替代方案等。

我们可以参考下面代码废弃函数的注释：

```
com.google.common.io.LittleEndianDataOutputStream#writeBytes
/**
 * @deprecated The semantics of {@code writeBytes(String s)} are considered dangerous. Please use
 *     {@link #writeUTF(String s)}, {@link #writeChars(String s)} or another write method instead.
 */
@Deprecated
@Override
public void writeBytes(String s) throws IOException {
  ((DataOutputStream) out).writeBytes(s);
}
```

该函数给出了废弃的原因：该函数比较危险。

给出了两个替代方案： `{@link #writeUTF(String s)}, {@link #writeChars(String s)}` 。

从这里我们学到，除了交代废弃的原因和替代方法外，还可以使用 `{@link}` 提供跳转到替代函数的快捷方式。



##### 3.4 警告类注释

比如有很多程序员为了方便测试会写一个测试控制器，如 `TestController` ，来提供 HTTP 接口的控制器，预留一些 “测试后门”，通常会有一个比较好的做法是放到某个特定测试分支，不会带到线上。

如：

- 提供查看项目的 apollo 配置项是否生效的接口。
- 提供查看 redis 数据的接口。
- 提供修复数据的接口。
- 提供某项功能的开关接口。
- 等

那么如果有些接口操作姿势 “非常特别” 或者 “非常危险”，一定要接口上加上注释，防止其他人员误触，导致故障。

如果某个函数仅供内部使用或者仅供某个功能使用，最好可以在注释上加上警示。

这些都极大降低沟通成功，极大降低团队其他成员犯错的几率。



##### 3.5 特殊注释

开发中特殊注释如：TODO 注释和 FIXME 注释也非常常见。

**TODO 注释主要用在本该做还没做的事项。**

- 待斟酌函数的命名。
- 性能不佳，待后期优化。
- 开发过程某个功能使用前需要进行权限校验，但是权限校验依赖的新接口对方还没开发好。

此时可以加上 TODO 注释可以参考下面格式：

```java
// TODO:: [xxx功能] 负责人：张三，事项：待添加权限验证逻辑，添加时间：2019-08-25 预计处理时间：2019-09-01
```

包含功能名称、责任人、事项、添加时间和预处理时间等信息。

我们看看 `com.google.common.io.Resources#getResource(java.lang.String)` 源码：

```java
/**
 * Returns a {@code URL} pointing to {@code resourceName} if the resource is found using the
 * {@linkplain Thread#getContextClassLoader() context class loader}. In simple environments, the
 * context class loader will find resources from the class path. In environments where different
 * threads can have different class loaders, for example app servers, the context class loader
 * will typically have been set to an appropriate loader for the current thread.
 *
 * <p>In the unusual case where the context class loader is null, the class loader that loaded
 * this class ({@code Resources}) will be used instead.
 *
 * @throws IllegalArgumentException if the resource is not found
 */
@CanIgnoreReturnValue // being used to check if a resource exists
// TODO(cgdecker): maybe add a better way to check if a resource exists
// e.g. Optional<URL> tryGetResource or boolean resourceExists
public static URL getResource(String resourceName) {
  ClassLoader loader =
      MoreObjects.firstNonNull(
          Thread.currentThread().getContextClassLoader(), Resources.class.getClassLoader());
  URL url = loader.getResource(resourceName);
  checkArgument(url != null, "resource %s not found.", resourceName);
  return url;
}
```

其中注释的最后两行用到了 TODO 注释，该注释包含了责任人和修改思路。

因此如果有未来优化的思路时，可以通过 TODO 进行注释，在未来代码迭代时实现该注释的想法。

`com.google.common.escape.Escapers#wrap` 也有一个不错的范例：

```java
private static UnicodeEscaper wrap(final CharEscaper escaper) {
  // 省略
   if (hiChars != null) {
   // TODO: Is this faster than System.arraycopy() for small arrays?
    for (int n = 0; n < hiChars.length; ++n) {
      output[n] = hiChars[n];
    }
  } else {
    output[0] = surrogateChars[0];
  }
  // 省略
}
```

这里表明作者还没有将两者性能进行对比，得到最佳选项。

**FIXME 注释，主要用在某些出错代码处，一般是一些不能工作需要及时纠正的错误。**

如编写了一处代码，其中部分代码涉及到了计算，但是自测时发现计算结果出错。此时可以参考下面的格式添加 FIXME 注释。在代码上线前一定要修复并验证好相关错误。

示例：

```
// FIXME:: [xxx功能] 负责人：张三，错误：计算错误，添加时间：2019-08-08 预计处理时间：2019-09-01
```



#### 4. 为什么这么规定？

不知道大家有没有思考过这个问题： **为什么《手册》会有这些规定？**

我想这么做的最主要原因是为了帮助读代码的人员快速理解代码。

下面选取几条进行解读：

**【强制】方法内部单行注释，在被注释语句上方另起一行，使用 // 注释。方法内部多行注释**
**使用 /\* */ 注释，注意与代码对齐。**

方法内部单行注释，在被注释的语句上方另起一行。主要体现了整体思维，也是为了实现 ” 代码意群 “效应，从视觉上让注释和下面的代码更接近。

**【强制】 所有的类都必须添加创建者和创建时间。**

类添加了创建者，读者就可以知道第一个创建该类的人（一般是最熟悉的人）是谁，遇到问题可以找他核实。

类添加了创建时间，有助于阅读此代码的人更方便地了解类的编写时间。

另外在这里给出一个技巧：如果我们使用的是 IDEA，并用 GIT 进行代码版本管理，可以在编辑器的左侧行数附近，右键选择 “Annotate”， 可以查看某行代码修改的人和时间。

如果你对该部分代码有疑问，可以快速定位到修改的人和修改时间，对我们协调和解决问题有极大的帮助。



#### 5 补充

**【强制】如果代码逻辑和注释不符，必须进行修改**

代码逻辑和注释不符，容易让使用者误用，增加出错的概率，容易造成返工降低开发效率。

通常由于开发者理解有误，偶尔会写出了误导性注释，如果发现这类问题一定要认真核实，如果确认是误导性注释，一定要及时修改，避免团队其他成员重复趟坑。

**【推荐】 TODO 注释要加上功能名称**

**为什么特殊注释要加上功能名称？**

通常我们会有很多项目的 TODO 注释，但是最常遇到的需求是快速定位正在开发的某个功能的 TODO 注释或者其他某个想修改的功能的 TODO 进行修改。此时如果 TODO 较多且没有标注功能名称，要想找到自己要修改的 TODO 项，通常需要通过搜索自己的姓名来查找，如果 TODO 较多查找起来将非常耗时。

**【推荐】方法注释中建议添加相关需求文档，接口文档地址。**

很多公司都会有接口平台，开发人员可以将 Dubbo 或 HTTP 接口传到接口平台中，从而得到访问链接，方便前后端对接。

建议将上传的 Dubbo 和 HTTP 接口文档地址顺手加入到注释中，避免每次需要使用时都要手动搜索。

```java
/**
 *  xxx功能（功能描述）
 *
 *  需求文档：{@link <a href="http://doc.imooc.com/xxx/process/0001"/>}
 *  接口文档：{@link <a href="http://api.imooc.com/xxx/process/0001"/>}
 *  对接人员：@张三
 *
 * @param param  参数描述
 * @return  返回值描述
 */
public Object something(Object param){
    // 1. 查询xx数据
    
    // 2. 过滤yy条件
    
    // 3. 组装结果
}
```

尤其是对依赖的三方 / 二方接口的封装，大家可以将此接口的相关文档和负责人添加到注释中。

后面自己也可能经常需要找接口的文档链接，开发过程中遇到问题也可及时和对接人沟通，这将极大提供工作效率。

这是一条非常值得推荐的技巧，这种注释风格非常能够体现出一个人的专业素养。

**【推荐】容易费解的地方一定要加注释。**

自己某块代码的写法很诡异，一定要注明原因。

否则极有可能因为时间久远，后面自己再回头看，或者别人问你为什么这么写，自己都蒙圈了。

导致别人不敢乱改，自己也不敢改动的尴尬情况。

这将是一个非常大的隐患。

**【推荐】推荐 git 提交注释的格式为： [功能名称] < 提交类型 > 修改点描述。**

很多公司对 git 提交注释的格式有自己的要求， 但是很多公司没有规定，导致大家写的都很随意。

很多人提交的注释都是功能的描述，无法得知因哪个功能做的修改。

建议大家可以养成好的习惯，在提交的描述中增加功能名称，并且能够再添加修改的性质就更好了。

修改的性质包括：**新增、删除、修改、修复等**。

比如我们独立开发的一个功能，突然中间有一个提交没有带我们的功能名称或功能名称不对，我们可以及时感知到可能出现了问题。

比如我们很久之后发现之前自己对某个函数进行了修改，自己却忘记修改的目的，我们可以查看提交记录，根据注释快速了解到是由于哪个功能导致的修改。

**正例**：

```
[a功能] <add>  某某接口

[a功能] <delete> 删除了无用的注释

[a功能] <update>  修改函数命名

[a功能] <fix> 修复了某个错误
```

大家实践之后就会发现该规约的好处。

**【参考】 利用 //----- 或 /\* ---- 分组 — \*/ 注释实现” 方法分组 “**

`org.apache.commons.lang3.BooleanUtils` 工具类中就广泛应用了这种方式：

```java
// Integer to Boolean methods
//-----------------------------------------------------------------------
    
// 各种整型转布尔类型的函数
```

再如 `java.util.HashMap` 中的方法分组注释:

```java
/* ---------------- Static utilities -------------- */
```

通过方法分组的注释可以很好地实现函数的 ” 分组 “，将类中功能相似的方法放在一起，并使用上述注释进行分割，是一个不错的技巧。

**【参考】多写设计的目的，注意事项，不要写从代码显而易见的注释。**

很多人喜欢写一些显而易见的注释，导致自己花费了时间对团队其他人却没太大帮助。

如果方法比较复杂，尽量写**设计的目的和注意的事项等更有帮助的内容**。

**反例**：

```java
public Boolean isLegal() {
    // 如果在售或有库存或有敏感词则返回false
    if (isOnSell == null || hasStock == null || hasSensitiveWords == null) {
        return false;
    }
    return isOnSell && hasStock && (!hasSensitiveWords);
}
```

**【参考】 可以将方法的核心逻辑拆分成多个步骤，关键步骤在函数内部可以加上注释并带上序号，之前空一行。**

函数内的逻辑注释，将有助于我们养成任务拆解的思维，也有助于自己或团队其他成员快速理解编程的逻辑。

如果核心逻辑的关键步骤加上注释，当代码较长时可以快速帮助读代码的人理解。

这样当代码行数超过 80 行时，开发者也可以根据核心逻辑注释来拆分子函数。

即使不在核心步骤添加注释 (或提取子函数)，在核心步骤之间加上一个空格行，方便读者理解。

大家可以在每个大的步骤前加注释，也可以在核心逻辑前将注释分条列举。

可以参考 `java.util.concurrent.ThreadPoolExecutor#execute` 函数的注释：

```java
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```



#### 6. 总结

本节如果你只记一句话那就是：**注释的目的是让读者更快理解代码的含义** 。注释的其他规约都是围绕这一点展开的。

本节讲述了注释的目的，并结合实际的开发经验对注释相关规约进行了解读和补充。

编写恰当的注释是一个程序员专业性的体现，希望大家在编程中能够严格要求自己，能够认真实践好的注释规范，提高开发效率，少趟坑。



### 你真得了解可变参数吗?



#### 1.前言

《手册》第 7 页 有一段关于 Java 变长参数的规约[1](https://www.imooc.com/read/55/article/1150#fn1) ：

> 【强制】相同参数类型，相同业务含义，才可以使用 Java 的可变参数，避免使用 `Object` 。说明:可变参数必须放置在参数列表的最后。(提倡同学们尽量不用可变参数编程)
> 正例: `public List listUsers(String type, Long... ids) {...}`

那么我们要思考下面几个问题：

- 为什么要有变长参数？
- 可变参数的常见用法是什么？
- 可变参数有哪些诡异的表现？

本节将详细探讨这些问题。



#### 2 变长参数的思考



##### 2.1 初步了解可变参数

我们知道可变参数（vararg）方法（又叫 variable arity method）语言特性是在 Java 5 出现的。

可变参数方法接受 **0 到多个相同类型参数（通常都是1个及以上）**。

其核心原理是：**创建一个数组，数组大小为可变参数传入的元素个数，最终将数组传递给方法**。



##### 2.2 可变参数的思考

我们学习 Java 一些语言特性时，最好能够思考它为什么会出现？是为了解决什么问题？有哪些优势？没有它会有哪些困难？等。

我们思考这样一个问题：**可变参数的目的是什么?**

试想一下，如果没有变长参数的语言特性，我们会怎么处理？

- 我们可以通过定义多个相同类型的参数进行重载。但是如果这样做如果参数数量不固定就无法实现。
- 我们还可以通过定义数组的参数进行重载。但是这就要求调用时要构造数组，又变成了 “定长”，而且需要增加构造数组的代码，代码不够简洁。

由此可见，变长参数适应了不定参数个数的情况，**避免了手动构造数组，提高语言的简洁性和代码的灵活性**。



#### 3.常见变长参数函数



##### 3.1 JDK中变长参数函数举例

包括 JDK 在内的很多库都封装了很多带有变长参数的函数。

`java.lang.String#format(java.lang.String, java.lang.Object...)` 就是JDK 中非常常见的变长参数函数之一。

其源码如下：

```java
  /**
     * Returns a formatted string using the specified format string and
     * arguments.
     *
     * <p> The locale always used is the one returned by {@link
     * java.util.Locale#getDefault() Locale.getDefault()}.
     *
     * @param  format
     *         A <a href="../util/Formatter.html#syntax">format string</a>
     *
     * @param  args
     *         Arguments referenced by the format specifiers in the format
     *         string.  If there are more arguments than format specifiers, the
     *         extra arguments are ignored.  The number of arguments is
     *         variable and may be zero.  The maximum number of arguments is
     *         limited by the maximum dimension of a Java array as defined by
     *         <cite>The Java&trade; Virtual Machine Specification</cite>.
     *         The behaviour on a
     *         {@code null} argument depends on the <a
     *         href="../util/Formatter.html#syntax">conversion</a>.
     *
     * @throws  java.util.IllegalFormatException
     *          If a format string contains an illegal syntax, a format
     *          specifier that is incompatible with the given arguments,
     *          insufficient arguments given the format string, or other
     *          illegal conditions.  For specification of all possible
     *          formatting errors, see the <a
     *          href="../util/Formatter.html#detail">Details</a> section of the
     *          formatter class specification.
     *
     * @return  A formatted string
     *
     * @see  java.util.Formatter
     * @since  1.5
     */
    public static String format(String format, Object... args) {
        return new Formatter().format(format, args).toString();
    }
```

根据参数名称或源码注释可知：第一个参数是格式定义，第二个参数为变长参数为前面的格式定义占位符对应的参数。

用法如下：

```java
@Test
public void format() {
    String pattern = "我喜欢在 %s 上学习 %s";
    String arg0 = "https://www.imooc.com/";
    String arg1 = "编程";
    String format = String.format(pattern, arg0, arg1);

    String expected = "我喜欢在 " + arg0 + " 上学习 " + arg1;
    Assert.assertEquals(expected, format);
}
```

由于第二个参数为变长参数，我们只需要根据前面占位符的个数填充对应个数的参数即可，非常方便。



##### 3.2 第三方库的可变参数函数举例

再如 commons-lang3 的字符串工具类 `org.apache.commons.lang3.StringUtils#isAllEmpty`函数源码：

```java
/**
 * <p>Checks if all of the CharSequences are empty ("") or null.</p>
 *
 * <pre>
 * StringUtils.isAllEmpty(null)             = true
 * StringUtils.isAllEmpty(null, "")         = true
 * StringUtils.isAllEmpty(new String[] {})  = true
 * StringUtils.isAllEmpty(null, "foo")      = false
 * StringUtils.isAllEmpty("", "bar")        = false
 * StringUtils.isAllEmpty("bob", "")        = false
 * StringUtils.isAllEmpty("  bob  ", null)  = false
 * StringUtils.isAllEmpty(" ", "bar")       = false
 * StringUtils.isAllEmpty("foo", "bar")     = false
 * </pre>
 *
 * @param css  the CharSequences to check, may be null or empty
 * @return {@code true} if all of the CharSequences are empty or null
 * @since 3.6
 */
public static boolean isAllEmpty(final CharSequence... css) {
    if (ArrayUtils.isEmpty(css)) {
        return true;
    }
    for (final CharSequence cs : css) {
        if (isNotEmpty(cs)) {
            return false;
        }
    }
    return true;
}
```

该函数的功能是判断传入的参数（个数不固定）是否都是空字符串或 `null`。

用法非常简单：

```java
@Test
public void isAllEmpty(){
    boolean result = StringUtils.isAllEmpty(null, "foo");
    Assert.assertFalse(result);
}
```

**有了变长参数支持，我们不需要根据参数的数量构造定长数组或变长的集合，用法上更加简洁**。

我们还看到`org.apache.commons.lang3.StringUtils` 工具类中还封装了

`StringUtils#isEmpty` 单个参数的判空函数。

通过函数命名和参数列表可以很容易地区分哪个是针对单参数，哪个是针对多参数（变长参数）。

这里也隐含了一个潜规则： 虽然变长参数支持 0 到多个参数，但是更多时候是用在 2 个参数及其以上的场景。

大家编写带变长参数函数时可以借鉴这种写法，即为单个参数和不定数量参数编写两个不同的函数。

如果大家平时使用三方工具包时能够留心看其源码，还会发现很多类似的变长参数函数。



#### 4.可变参数诡异问题分析

通过上面的两个例子，我们了解了变长参数函数的优势。

接下来我们通过下面一个示例并结合 commons-lang 包的布尔工具类： `org.apache.commons.lang3.BooleanUtils` 来学习和分析可变参数导致的一个诡异问题。

示例代码：

```java
public class BooleanDemo {

    public static void main(String[] args) {
        boolean result = and(true, true, true);
        System.out.println(result);
        justPrint(true);
    }

  // 函数1
    private static void justPrint(boolean b) {
        System.out.println(b);
    }

  // 函数2
    private static void justPrint(Boolean b) {
        System.out.println(b);
    }

  // 函数3
    private static boolean and(boolean... booleans) {
        System.out.println("boolean");
        for (boolean b : booleans) {
            if (!b) {
                return false;
            }
        }
        return true;
    }

 // 函数4
    private static boolean and(Boolean... booleans) {
        System.out.println("Boolean");
        for (Boolean b : booleans) {
            if (!b) {
                return false;
            }
        }
        return true;
    }
}
```

请问上面程序的结果是什么呢？

相信很多人会回答 `true`、`true`。

回答的依据应该是：示例中 `main` 函数调用的可变参数都是基本类型，因此和函数 3 最贴合，应该会选择函数 3 来执行。

实际是这样的吗？

将代码输入到 IDEA，就会发现 IDEA 就会给出下面这段提示：

> Ambiguous method call. Both `and (boolean...)` in `BooleanDemo` and `and (Boolean...)` in `BooleanDemo` match.
>
> 模糊的函数调用。该函数调用和 `and (boolean...)` 和 `and (Boolean...)`两个函数签名都匹配。



##### 4.1 为啥会提示 ambiguous method call ？

很多人看到这里可能会毫无头绪，**我们该怎么学习和分析这个问题呢？**

按照我们的传统，我们从 JLS[2](https://www.imooc.com/read/55/article/1150#fn2)中寻找答案。 我们发现其中 [15.12.2 节 Compile - Time Step 2 : Determine Method Signature](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.12.2) 中提到：

> 为了兼容Java SE 5.0 之前的版本，方法签名的选择分为 3 个阶段。
>
> 第一阶段：**不让自动装箱和拆箱，也不能使用可变参数的情况下选择重载**。如果无法选择合适地方法，则进入第二阶段。
>
> 由于不允许自动拆箱、拆箱和可变参数，这一条保证了Java SE 5.0 之前的函数调用的合法性。
>
> 如果在第一阶段可变参数生效，如果在一个已经声明了 `m(Object)` 函数的类中声明 `m(Obejct...)` 函数，会导致即使有更适合的表达式（如 `m(null)` ） 也不会选择 `m(Object)` 。
>
> 第二阶段：**允许自动装箱和拆箱，但是仍然排除变长参数的重载**。如果仍然无法选择合适的方法，则进入第三阶段。
>
> 这是为了保证，如果定义了定长参数的函数情况下，不会选择变长参数。
>
> 第三阶段：**允许自动装箱、拆箱和变长参数的重载**。

因此可见，在选择函数签名时，有以下几个阶段：
![图片描述](img/5dcb66570001560f15140336.png)

我们再回头看下示例代码。

第一阶段，选择了函数1。

第二阶段，允许自动装箱和拆箱，但是仍然不匹配可变参数的函数，仍然无法确认使用哪个 `and`函数，因为自动装箱仍然没有找到 3 个 boolean 参数的 `and` 函数。

第三阶段，允许自动装箱和拆箱，允许匹配变长参数。

问题就出现在第三个阶段，允许匹配变长参数时就要允许自动拆箱和装箱，这样函数 3 和函数 4 都可匹配到，因此无法通过编译。



##### 4.2 变长参数的本质是什么？



###### 4.2.1 反编译

我们对项目进行编译，来到 IDEA的 target 目录，查看编译后的 class 文件。

也可以直接用 `javac BooleanDemo.java` 对该类进行编译，然后通过前面介绍的 JD-GUI 反编译工具查看。

下面是反编译后的代码：

```java
// 函数3
private static boolean and(boolean... booleans) {
        System.out.println("boolean");
        boolean[] var1 = booleans;
        int var2 = booleans.length;

        for(int var3 = 0; var3 < var2; ++var3) {
            boolean b = var1[var3];
            if (!b) {
                return false;
            }
        }

        return true;
    }

// 函数4
private static boolean and(Boolean... booleans) {
      System.out.println("Boolean");
      Boolean[] var1 = booleans;
      int var2 = booleans.length;

      for(int var3 = 0; var3 < var2; ++var3) {
          Boolean b = var1[var3];
          if (!b) {
              return false;
          }
      }

      return true;
}
```

我们可以清楚地看到，变长参数编译后内部通过数组来处理。



###### 4.2.2 调试

我们还可以在函数 3 中打断点，来观察 `booleans` 这个参数对象的各种属性。

![图片描述](img/5dcb66460001c58f09210577.png)
通过 “variables” 可预览到参数的类型和数据，可以看到 `booleans` 为 `boolean` 类型的数组，长度为 3。

我们还可以通过在 “variables” 选项卡的`booleans` 上右键，选择 “Evaluate Expression”, 然后通过调用 `booleans.getClass().isArray()` 来验证其是否为数组，查看其长度等。

未来有类似的场景，大家都可以通过断点调试来观察数据，还可以通过表达式来研究对象的一些属性。

更多高级的调试技巧请参考本专栏后续章节。



##### 4.3 如何解决？

我们如果使用 commons-lang3 的 `org.apache.commons.lang3.BooleanUtils` 工具类中 `and` 函数，也会遇到类似的错误。

下面源码取自 commons-lang3 的 3.9版本。

```xml
<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.9</version>
</dependency>
```

该类中有两个重载的变长参数函数：

```
org.apache.commons.lang3.BooleanUtils#and(boolean...)
 /**
     * <p>Performs an and on a set of booleans.</p>
     *
     * <pre>
     *   BooleanUtils.and(true, true)         = true
     *   BooleanUtils.and(false, false)       = false
     *   BooleanUtils.and(true, false)        = false
     *   BooleanUtils.and(true, true, false)  = false
     *   BooleanUtils.and(true, true, true)   = true
     * </pre>
     *
     * @param array  an array of {@code boolean}s
     * @return {@code true} if the and is successful.
     * @throws IllegalArgumentException if {@code array} is {@code null}
     * @throws IllegalArgumentException if {@code array} is empty.
     * @since 3.0.1
     */
    public static boolean and(final boolean... array) {
        // Validates input
        if (array == null) {
            throw new IllegalArgumentException("The Array must not be null");
        }
        if (array.length == 0) {
            throw new IllegalArgumentException("Array is empty");
        }
        for (final boolean element : array) {
            if (!element) {
                return false;
            }
        }
        return true;
    }

 
```

`org.apache.commons.lang3.BooleanUtils#and(java.lang.Boolean...)` 的源码和注释如下：

```java
 /**
     * <p>Performs an and on an array of Booleans.</p>
     *
     * <pre>
     *   BooleanUtils.and(Boolean.TRUE, Boolean.TRUE)                 = Boolean.TRUE
     *   BooleanUtils.and(Boolean.FALSE, Boolean.FALSE)               = Boolean.FALSE
     *   BooleanUtils.and(Boolean.TRUE, Boolean.FALSE)                = Boolean.FALSE
     *   BooleanUtils.and(Boolean.TRUE, Boolean.TRUE, Boolean.TRUE)   = Boolean.TRUE
     *   BooleanUtils.and(Boolean.FALSE, Boolean.FALSE, Boolean.TRUE) = Boolean.FALSE
     *   BooleanUtils.and(Boolean.TRUE, Boolean.FALSE, Boolean.TRUE)  = Boolean.FALSE
     * </pre>
     *
     * @param array  an array of {@code Boolean}s
     * @return {@code true} if the and is successful.
     * @throws IllegalArgumentException if {@code array} is {@code null}
     * @throws IllegalArgumentException if {@code array} is empty.
     * @throws IllegalArgumentException if {@code array} contains a {@code null}
     * @since 3.0.1
     */
    public static Boolean and(final Boolean... array) {
        if (array == null) {
            throw new IllegalArgumentException("The Array must not be null");
        }
        if (array.length == 0) {
            throw new IllegalArgumentException("Array is empty");
        }
        try {
            final boolean[] primitive = ArrayUtils.toPrimitive(array);
            return and(primitive) ? Boolean.TRUE : Boolean.FALSE;
        } catch (final NullPointerException ex) {
            throw new IllegalArgumentException("The array must not contain any null elements");
        }
    }
```

错误的原因和前面的示例所分析的一致，都是在选择函数签名时，在前两个阶段没找到匹配的函数，允许变长参数匹配时，允许自动装箱和拆箱，却找到了两个可以匹配的函数。

我们如果直接参考两个工具函数注释上的例子，会发现编译无法通过。从这一点来看，如果注释中的用法和实际使用无法对应，会对使用者造成极大地困扰。

**那么到底如何解决这个问题呢？**

正如前面讲到的，**我们可以看源码的单元测试，也可以通过 codota 来学习其他优秀的开源项目关于此函数的用法。**

接下来我们实践一下。



###### 4.3.1 查看源码的单元测试

我们拉取 [commons-lang](https://github.com/apache/commons-lang) 源码，找到了 `BooleanUtilsTest` 关于 `and` 函数相关的单元测试代码。

```
org.apache.commons.lang3.BooleanUtilsTest#testAnd_primitive_validInput_2items
    @Test
    public void testAnd_primitive_validInput_2items() {
        assertTrue(
                BooleanUtils.and(new boolean[] { true, true }),
                "False result for (true, true)");

        assertTrue(
                ! BooleanUtils.and(new boolean[] { false, false }),
                "True result for (false, false)");

        assertTrue(
                ! BooleanUtils.and(new boolean[] { true, false }),
                "True result for (true, false)");

        assertTrue(
                ! BooleanUtils.and(new boolean[] { false, true }),
                "True result for (false, true)");
    }
// 省略其他
```

通过单元测试的代码，我们发现相关的测试代码的参数都是通过数组传入。

`org.apache.commons.lang3.BooleanUtils#and(java.lang.Boolean...)` 相关的单测亦然。

因此我们可以放弃“变长参数”的好处，“回归自然”，我们可以仿照类似写法，使用数组传参。



###### 4.3.2 codota大法

我们在 [codota](https://www.codota.com/code) 上找到该函数的相关范例，可以很好地解决本节所提到的问题。

第一个范例是自定义工具类来包装 `org.apache.commons.lang3.BooleanUtils#and(boolean...)` 函数：
![图片描述](img/5dcb661d000112a508770215.png)

因为此工具类只包装了其中基本类型变长函数，如果传入基本类型的变长参数可以匹配，如果传入包装类型可以在第二阶段拆箱匹配到该工具函数。

第二个示例也是自定义工具类，但是参数是集合，实际使用时将集合转成数组再调用 `org.apache.commons.lang3.BooleanUtils#and(java.lang.Boolean...)`。
![图片描述](img/5dcb662b00018a5808650153.png)

通过该示例我们发现作者是用集合来替代不定长参数解决此问题的。

> 注：通过 codota 我们还可以看到该工具类的其他函数的一些常见用法。

以上两种方法都是通过自定义工具类的包装，巧妙地避免了直接调用该工具类导致函数签名选择的冲突问题。



#### 5.总结

本文主要介绍了变长参数的主要使用场景， 变长参数使用过程中的一个诡异问题，带着大家分析该问题背后的原因，并给出了解决该问题的方法。

希望大家遇到类似问题时，能够通过本文提供的方法来快速分析原因，并找到应对的办法。

下一节我们将讲述集合去重的正确姿势，会对不同去重方式的性能差异的原因进行分析，并对其性能进行对比。



#### 课后练习

- 结合之前空指针章节所讲的内容，思考示例程序有啥隐患？该如何避免呢？
- 结合本节学的内容，请封装一个工具类，包装`org.apache.commons.lang3.BooleanUtils#or(java.lang.Boolean...)` 函数，避免选择函数签名时的冲突问题。



### 集合去重的正确姿势



#### 1. 前言

《手册》的第 11 页 关于集合处理的章节有这样的描述 [1](https://www.imooc.com/read/55/article/1151#fn1)：

> 【参考】利用 Set 元素唯一的特性，可以快速对一个集合进行去重操作，避免使用 List 的 contains 方法进行遍历、对比、去重操作。
>
> 【强制】关于 hashCode 和 equals 的处理，遵循如下规则:
>
> 1. 只要覆写 equals，就必须覆写 hashCode；
> 2. 因为 Set 存储的是不重复的对象，依据 hashCode 和 equals 进行判断，所以 Set 存储的对象必须覆写这两个方法；
> 3. 如果自定义对象作为 Map 的键，那么必须覆写 hashCode 和 equals。
>    说明：String 已覆写 hashCode 和 equals 方法，所以我们可以愉快地使用 String 对象作为 key 来使用。

可能很多人会认为工作之后不会有人通过 `List` 的 `contains` 函数来去重，然而，最可怕的是真的有…

那么我们思考下面几个问题：

- Set 是怎样保证数据的唯一性呢？
- Set 存储的是不重复的对象，是不是根据 hashCode 和 equals 来判断是否重复的呢？
- Set 和 List 的去重性能差距是多大呢？

本节将重点研究这几个问题。



#### 2. 唯一性保证

开发中常见到使用 `Set` 去重的代码如下：

```java
public static <T> Set<T> removeDuplicateBySet(List<T> data) {

    if (CollectionUtils.isEmpty(data)) {
        return new HashSet<>();
    }
    return new HashSet<>(data);
}
```

> 注：Set 自身可以保证不重复，不需要先通过 contains 判断不存在再添加元素。

我们先看 `java.util.HashSet` 的类注释 （注释内容省略，具体请大家看源码）中的一些要点：

> 实现了 `Set` 接口。
>
> 基于 `HashMap` 来实现核心功能。
>
> 允许存入 `null` 元素，不保证顺序。
>
> 该类的方法并没同步，如果想要同步需要外部处理，可以构造一个同步对象，也可以使用 `Collections#synchronizedSet` , 最佳实践：
>
> ```
> Set s = Collections.synchronizedSet(new HashSet(...));
> ```
>
> 迭代方法返回的迭代器是 “fail-fast” 的，迭代器创建后如果调用除了迭代器自己的 `remove` 函数外的其他修改方法，会抛出：`ConcurrentModificationException`。

我们再看看 `HashSet` 对应的构造函数 `java.util.HashSet#HashSet(java.util.Collection)` 源码：

```java
/**
 * Constructs a new set containing the elements in the specified
 * collection.  The <tt>HashMap</tt> is created with default load factor
 * (0.75) and an initial capacity sufficient to contain the elements in
 * the specified collection.
 *
 * @param c the collection whose elements are to be placed into this set
 * @throws NullPointerException if the specified collection is null
 */
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
```

从这里我们看到了底层的确是通过 `HashMap` 支持的，根据参数集合的长度构造对应默认容量的 `HashMap`。

然后调用父类的 `java.util.AbstractCollection#addAll` （添加所有元素的函数）：

```java
  public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }
```

从这里可以看出，通过 `for-each` 语法糖对集合进行迭代并调用 `add` 函数将元素依次添加到 `HashSet` 中。

```java
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

/**
 * Adds the specified element to this set if it is not already present.
 * More formally, adds the specified element <tt>e</tt> to this set if
 * this set contains no element <tt>e2</tt> such that
 * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
 * If this set already contains the element, the call leaves the set
 * unchanged and returns <tt>false</tt>.
 *
 * @param e element to be added to this set
 * @return <tt>true</tt> if this set did not already contain the specified
 * element
 */
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

通过这个函数的注释，我们可以看到：

> 该函数的功能是添加 set 中没添加过的元素。
>
> 更准确地说，如果想将元素 e 添加到此集合中，那么集合中不能存在元素 e2 满足：
>
> **( e== null ? e2 ==null : e.equals(e2) )** 。
>
> 如果已经包含了该元素，那么集合将不会发生改变，将返回 false。

从这里我们还看到，为了保持 `HashMap` 的用法，这里给底层的 `Map` 的值传入一个傀儡对象（PRESENT）。

我们进入更底层源码 `java.util.HashMap#put` :

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

通过这里我们看到，除了传入 `key` 和 `value` 外，第一个哈希值的参数 (`hash`) 是通过 `HashMap` 的 `hash` 函数实现的。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可以看到如果 `key` 为 `null` ，哈希值为 0，否则将 `key` 通过自身 `hashCode` 函数计算的的哈希值和其右移 16 位进行异或运算得到最终的哈希值。

在 `java.util.HashMap#putVal` 中，直接通过 `(n - 1) & hash` 来得到当前元素在节点数组中的位置。如果不存在，直接构造新节点并存储到该节点数组的对应位置。如果存在，则通过下面逻辑：

```java
p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))
```

**来判断元素是否相等。**

**如果相等则用新值替换旧值，否则添加红黑树节点或者链表节点。**

这就是前言中第 2 和第 3 条规定的依据。

最终如果存在 `key` 则返回旧值，不存在则返回 `null`。

此时，我们再回看 `java.util.HashSet#add` 源码:

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

一切就非常清晰了。

通过 `HashMap` 的 `key` 的唯一性保证 `HashSet` 的元素的唯一性。

我们再看 `HashSet` 的迭代器 `java.util.HashSet#iterator`:

```java
 /**
     * Returns an iterator over the elements in this set.  The elements
     * are returned in no particular order.
     *
     * @return an Iterator over the elements in this set
     * @see ConcurrentModificationException
     */
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
```

我们发现，其实 `HashSet` 的元素是存放在 `HashMap` 的 `keySet` 中。

大家可以进入 `HashSet` 的其他方法中查看，可以发现几乎 `HashSet` 的所有核心函数都是通过 `HashMap` 支撑的。

**由于 `HashSet` 底层采用 `HashMap` 实现，通过上述分析，我们可知其 “去重” 的时间复杂度是 O (n)。**

另外我们回答前言中 “关于 hashCode 和 equals 的处理” 的第 1 条：** 只要覆写 equals，就必须覆写 hashCode”。** 这个问题。

除了上面讲到的判断重复的依据外，从其源码 `java.lang.Object#equals` 的注释中也可以得到更本质的原因：

> Note that it is generally necessary to override the {@code hashCode} method whenever this method is overridden, so as to maintain the general contract for the {@code hashCode} method, which states that equal objects must have equal hash codes.
>
> 只要重写 equals 方法就要重新 hashCode 方法，来维持 hashCode 的约定，即 **equals 的对象的哈希值必须相等**。



#### 3. 性能差异的原因

前面讲到 “由于 `HashSet` 底层采用了 `HashMap` 实现，因此去重的时间复杂度是 O (n)”。

那么，通过 `List` 的 `contains` 函数来去重，原理又是怎样的呢？时间复杂度是多少呢？

且看下面基于 `List` 的 `contains` 函数来去重示例代码：

```java
public static <T> List<T> removeDuplicateByList(List<T> data) {

    if (CollectionUtils.isEmpty(data)) {
        return new ArrayList<>();

    }
    List<T> result = new ArrayList<>(data.size());
    for (T current : data) {
        if (!result.contains(current)) {
            result.add(current);
        }
    }
    return result;
}
```

其实 `HashSet` 和 `ArrayList` 去重性能差异的核心在于 `contains` 函数性能对比。

我们分别查看 `java.util.HashSet#contains` 和 `java.util.ArrayList#contains` 的实现。

`java.util.HashSet#contains` 源码：

```java
/**
 * Returns <tt>true</tt> if this set contains the specified element.
 * More formally, returns <tt>true</tt> if and only if this set
 * contains an element <tt>e</tt> such that
 * <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>.
 *
 * @param o element whose presence in this set is to be tested
 * @return <tt>true</tt> if this set contains the specified element
 */
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

最终是通过 `java.util.HashMap#getNode` 来判断的（和 `java.util.HashMap#putVal` 的一些判断非常相似）：

```java
/**
 * Implements Map.get and related methods.
 *
 * @param hash hash for key
 * @param key the key
 * @return the node, or null if none
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

先通过计算过的 hash 值找到 table 对应索引的第一个元素进行比较，如果相等则返回第一个元素。

如果是树节点，从红黑树中查找该元素，否则在链表中查找该元素。

如果 hash 冲突不是极其严重（大多数都没怎么有哈希冲突），n 个元素依次判断并插入到 Set 的时间复杂度接近于 O (n)。

接下来我们看 `java.util.ArrayList#contains` 的源码：

```java
/**
 * Returns <tt>true</tt> if this list contains the specified element.
 * More formally, returns <tt>true</tt> if and only if this list contains
 * at least one element <tt>e</tt> such that
 * <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>.
 *
 * @param o element whose presence in this list is to be tested
 * @return <tt>true</tt> if this list contains the specified element
 */
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

其功能实现依赖于 `java.util.ArrayList#indexOf`:

```java
/**
 * Returns the index of the first occurrence of the specified element
 * in this list, or -1 if this list does not contain the element.
 * More formally, returns the lowest index <tt>i</tt> such that
 * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
 * or -1 if there is no such index.
 */
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

发现其核心逻辑为：如果为 null, 则遍历整个集合判断是否有 null 元素；否则遍历整个列表，通过 `o.equals(当前遍历到的元素)` 判断与当前元素是否相等，相等则返回当前循环的索引。

所以， n 个元素依次通过 `java.util.ArrayList#contains` 判断并插入到 Set 的时间复杂度接近于 O (n^2)。

因此，通过时间复杂度的比较，性能差距就不言而喻了。



#### 4. 性能对比

上面我们分别对性能的差异原因， 时间复杂度进行了分析。

我们分别将两个时间复杂度函数进行作图， 两者增速对比非常明显：
![图片描述](img/5dd219ff00016be606630420.png)

实践是检验真理的标准，因此我们写一段代码粗略对比一下他们的性能差异：

```java
@Slf4j
public class SetDemo {

    public static void main(String[] args) {

        List<Integer> lengthList = new LinkedList<>();
        int base = 1;
        for (int i = 1; i <= 6; i++) {
            base *= 10;
            lengthList.add(base);
        }

        StringRandomizer stringRandomizer = new StringRandomizer(10, 100, 1000);

        for (Integer length : lengthList) {
            log.debug("------------长度为 {} 时-------", length);
            ListRandomizer<String> listRandomizer = new ListRandomizer<>(stringRandomizer, length);
            List<String> data = listRandomizer.getRandomValue();

            StopWatch stopWatch = new StopWatch();
            stopWatch.start();
            Set<String> resultBySet = CollectionUtil.removeDuplicateBySet(data);
            log.debug("set去重耗时：{} ms", stopWatch.getTime());

            stopWatch = new StopWatch();
            stopWatch.start();
            List<String> resultByList = CollectionUtil.removeDuplicateByList(data);
            log.debug("list去重耗时：{} ms", stopWatch.getTime());
        }

    }
}
```

最终得到下面的数据：
![图片描述](img/5dd219ee000111d622520416.png)

我们重点观察最后两种情况：

> 长度为 10 万时使用 List 去重耗时接近 1 分钟，而使用 Set 去重则只需要 17 毫秒；
>
> 而集合长度为 100 万时，使用 List 去重，耗时则约为 1.7 小时，使用 Set 去重则只需要 1.33 秒。

对上述结果进行绘图如下：
![图片描述](img/5dd219df0001d40817101308.png)

通过此图，大家就可以非常直观地感受到两种去重方式的性能差异。

我们**发现当元素较少时两者耗时差距很小，随着元素的增多耗时差距越来越大**。

如果数据量不大时采用 `List` 去重勉强可以接受，但是数据量增大后，接口响应时间会超慢，这是难以忍受的，甚至造成大量线程阻塞引发故障。

> 在工作中一次排查慢接口时，查到了一个函数耗时较长，最终定位到是通过 List 去重导致的。
>
> 由于测试环境还有线上早期数据较少，这个接口的性能问题没有引起较大关注，后面频繁超时，才引起重视。

因此我们要养成好的编程习惯，尽可能地提高接口性能，避免因知识盲区导致故障。



#### 5. 为什么有人会这么用？

最后我们思考一下：为什么有人会用 List 的 contains 方法进行遍历、对比然后去重呢？

无非就是两个原因：

- 基础不扎实，不了解这种操作的时间复杂度；
- 为了维持返回值的类型。

对于第一个问题，基础不扎实我们要加强学习，多注意代码规范和代码的运行效率。

第二个问题是一种直线思维，是一种偷懒的表现。

比如某种特殊场景下需要的返回值类型为 `List`，“因此” 有些朋友就会声明一个 `List`，通过 `contains` 方法进行遍历、对比、去重，然后将其作为返回值返回。

其实，这种情况可以分两步走，先去重，然后通过 `ArrayList` 参数为集合的构造方法创建 `List` 对象来实现类型 “转换”，示例代码如下：

```java
// 数据
List<String> data = listRandomizer.getRandomValue();
// 先去重
Set<String> resultBySet = CollectionUtil.removeDuplicateBySet(data);

// 再转换格式
ArrayList<String> result = new ArrayList<>(resultBySet);
```



#### 6. 总结

本节主要讲述集合去重的正确姿势，主要要点有：

- `HashSet` 元素唯一性是通过 `HashMap` 的 key 唯一性来实现的；
- 性能的差距是元素查找函数的时间复杂度不同导致的；
- 元素较少时两者耗时差距很小，随着元素的增多耗时差距越来越大。



### 学习线程池的正确姿势



#### 1. 前言

《手册》第 14 页有关于线程池的论述 [1](https://www.imooc.com/read/55/article/1152#fn1)：

> 【强制】创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。
>
> 【强制】线程资源必须通过线程池提供，不允许在应用中自行显式创建线程
>
> 【强制】线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这 样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

看到这些规定我们可以思考下面几个问题：

- 那么为何会有这样的规定呢？
- 线程池那么重要，我们该如何学习线程池？

这些都是本节所要解决的问题。



#### 2. 规定解释



##### 2.1 为线程池指定有意义的线程名称

那么第一个问题：**为什么要指定有意义的线程名称呢？**

《手册》给出的解释是 “方便出错时回溯”。

如果大家还没啥体会的话， 可以对比一下下面通过 `jstack` 看到的线程片段：

默认命名：

```java
"pool-1-thread-1" #11 prio=5 os_prio=31 tid=0x00007fa0964c7000 nid=0x4403 waiting on condition [0x000070000db67000]

...

at java.lang.Thread.run(Thread.java:748)
```

自定义命名：

```java
"定时短息任务线程 thread-2" #11 prio=5 os_prio=31 tid=0x00007fa0964c7000 nid=0x4403 waiting on condition [0x000070000db67000]
...
at java.lang.Thread.run(Thread.java:748)
```

反差是不是很明显呢？

通过自定义名称，我们可以快速理解所关注的线程所属的线程池，对一些问题可以快速作出预判。

**如何实现呢？**

很多人总是先直接百度，直接查资料，虽然便捷，但是容易浅尝辄止，学啥都不深入，离开了资料就束手无策。

显然这不是我们想要的，那么怎么办呢？

我们可以去 `ThreadPoolExecutor` 的构造方法中寻找答案，构造函数中有一个 `threadFactory` 参数，通过常识或者其注释我们可以知道该参数是为线程池构造线程。

它的类型为： `java.util.concurrent.ThreadFactory` ，按照惯例，我们查看源码：

```java
/**
 * Constructs a new {@code Thread}.  Implementations may also initialize
 * priority, name, daemon status, {@code ThreadGroup}, etc.
 *
 * @param r a runnable to be executed by new thread instance
 * @return constructed thread, or {@code null} if the request to
 *         create a thread is rejected
 */
Thread newThread(Runnable r);
```

通过注释我们可以知道，重写此函数可以指定线程的优先级，设置是否是守护线程、设置线程的线程组等。

**那么我们如何找到自定义 `ThreadFactory` 的参考范例呢？**
![图片描述](img/5dd4dd47000130fc09190423.png)

大家可以通过点击左侧的 f 标志或使用快捷键查看实现类，进行学习。

具体写法我们可以参考：`net.sf.ehcache.util.NamedThreadFactory`

```java
/**
 * A {@link ThreadFactory} that sets names to the threads created by this factory. Threads created by this factory
 * will take names in the form of the string <code>namePrefix + " thread-" + threadNum</code> where <tt>threadNum</tt> is the 
 * count of threads created by this type of factory.
 * 
 * @author <a href="mailto:asanoujam@terracottatech.com">Abhishek Sanoujam</a>
 * 
 */
public class NamedThreadFactory implements ThreadFactory {

    private static AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
    private final boolean daemon;

    /**
     * Constructor accepting the prefix of the threads that will be created by this {@link ThreadFactory}
     * 
     * @param namePrefix Prefix for names of threads
     */
    public NamedThreadFactory(String namePrefix, boolean daemon) {
        this.namePrefix = namePrefix;
        this.daemon = daemon;

    }
    /**
     * Constructor accepting the prefix of the threads that will be created by this {@link ThreadFactory}
     *
     * @param namePrefix
     *            Prefix for names of threads
     */
    public NamedThreadFactory(String namePrefix) {
        this(namePrefix, false);
    }

    /**
     * Returns a new thread using a name as specified by this factory {@inheritDoc}
     */
    public Thread newThread(Runnable runnable) {
        final Thread thread = new Thread(runnable, namePrefix + " thread-" + threadNumber.getAndIncrement());
        thread.setDaemon(daemon);
        return thread;
    }

}
```

大家可以参考这个类进行改编。

另外，建议大家工作中如果不忙的时候要主动地去源码中看一看，看一些 JDK 源码中的接口有哪些实现类，它们的代码都是如何编写的，这对我们学习进阶有很大帮助。



##### 2.2 为什么不要在应用中显式创建线程？

这里要先讲一个设计模式：“对象池模式”， 参见 《Java 设计模式及实践》 34 页 [2](https://www.imooc.com/read/55/article/1152#fn2)：

> 对象的实例化是最耗费性能的操作之一，这在过去是一个大问题，现在已经不需要再那么关注。
>
> 但当我们处理封装外部资源的对象（例如数据库连接）时，对象的创建会耗费很多资源。
>
> 解决方案就是重用和共享这些创建成本昂贵的对象，这被称为对象池模式。

而根据《 Java 虚拟机规范 (Java SE 8 版)》第 9 - 15 页 [3](https://www.imooc.com/read/55/article/1152#fn3) 描述，以及《深入理解 Java 虚拟机：JVM 高级特性与最佳实践》第 39 页 [4](https://www.imooc.com/read/55/article/1152#fn4) 相关描述可知：
![图片描述](img/5dd4dd580001f25d14160888.png)

线程的创建需要开辟虚拟机栈、本地方法栈、程序计数器等线程私有的内存空间。

线程销毁时也会回收这些系统资源，因此如果频繁创建和销毁线程将大量消耗系统资源。

从上述特点我们可以看出，该场景非常符合对象池设计模式，其**核心目的是复用资源消耗加大的对象**。

> 建议大家学习设计模式时，着重了解设计模式的常见使用场景，优势和劣势，而不是着急跟着书上敲代码。
>
> 这样才能在看到对应的源码时能够 “恍然大悟”，遇到使用的场景时才能够想到要用这种设计模式。

另外，既然不提倡某种用法而提倡另外一种用法 / 技术，我们要着重思考另外一种用法 / 技术的优势。

不提倡手动创建线程的另外一个原因是线程池自身的优点，使用线程池有利于控制最大并发数，可以实现任务队列的缓存和拒绝策略，实现定时和周期执行任务，可以更好地隔离不同的场景等。



##### 2.3 如何学习线程池？

推荐通过源码和写 DEMO 来学习线程池。

那么为什么推荐这种学习方式呢？

这是因为：

1. 源码最权威，通过读源码印象更深刻，面试时或者使用时更有底气。
2. 写 DEMO 能够构造更多场景，我们可以通过运行看结果，通过各种调试技巧等方式验证自己的想法。

另外大家如果细心，可以看到很多人用过线程池，但是面试时或者和别人交流时迷迷糊糊。

为什么呢？

其实，这是因为很多人都是通过读书来记住线程池的一些参数和用法，而不是通过读源码和练习来学习的，导致印象不深刻，回答问题时没底气，没把握。

接下来我们介绍一下这两种不错的方式在线程池学习中的运用。



###### 2.3.1 读源码

我们先从 `java.util.concurrent.ThreadPoolExecutor` 的构造函数说起。

前面注释章节讲过 JDK 的注释是我们学习的典范。**我们看源码时注释是帮助我们理解的一大突破口。**

**如果不看书，我们如何更准确地理解参数含义呢？如何避免被一些博客误导呢？**

我们应该先从核心参数的名称对参数的含义有一个大概的了解，然后看再看线程池的核心函数的逻辑。

```java
/**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters.
 *
 * @param corePoolSize the number of threads to keep in the pool, even
 *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
 * @param maximumPoolSize the maximum number of threads to allow in the
 *        pool
 * @param keepAliveTime when the number of threads is greater than
 *        the core, this is the maximum time that excess idle threads
 *        will wait for new tasks before terminating.
 * @param unit the time unit for the {@code keepAliveTime} argument
 * @param workQueue the queue to use for holding tasks before they are
 *        executed.  This queue will hold only the {@code Runnable}
 *        tasks submitted by the {@code execute} method.
 * @param threadFactory the factory to use when the executor
 *        creates a new thread
 * @param handler the handler to use when execution is blocked
 *        because the thread bounds and queue capacities are reached
 * @throws IllegalArgumentException if one of the following holds:<br>
 *         {@code corePoolSize < 0}<br>
 *         {@code keepAliveTime < 0}<br>
 *         {@code maximumPoolSize <= 0}<br>
 *         {@code maximumPoolSize < corePoolSize}
 * @throws NullPointerException if {@code workQueue}
 *         or {@code threadFactory} or {@code handler} is null
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

通过注释我们可以清晰地知道每个参数的含义。

- `corePoolSize` 表示核心常驻线程池。即使空闲也会在线程池中保活，除非设置了允许核心线程池超时；
- `maximumPoolSize` 表示线程池同时执行的最大线程数量；
- `keepAliveTime` 表示线程池中的线程空闲时间，线程在销毁前等待新任务的最大时限；
- `unit` 表示 `keepAliveTime` 的单位；
- `workQueue` 存放执行前的任务。只会存放通过 `execute` 函数提交的 `Runnable` 任务；
- `threadFactory` 创建新线程的工厂；
- `handler` 线程超限且队列容量也达到最大值时执行受阻的处理策略。

注释中还给出了抛出异常的条件，大家可以自行学习。

接下来我们查看其核心函数之一的 `execute` 源码：

```java
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

通过注释我们可以知道，该函数的作用：

> 在未来的某个时刻执行给定的任务。该任务可能会被新创建的线程执行，也可能会被线程池中已经存在的线程执行。
>
> 如果任务因为 executor 被关闭 (shutdown) 或者容量达到上限而不能再提交执行时，会调用当前设置的 `RejectedExecutionHandler`。

另外源码中关于执行步骤的注释是我们理解线程池的关键：

> `execute` 分为 3 个处理步骤：
>
> 1、如果线程池中小于 `corePoolSize` 个执行的线程，则新建线程将当前任务作为第一个任务来执行；
>
> 2、如果任务成功入队，我们仍然需要 double-check 判断是否需要往线程池中新增线程（因为上次检查后可能有一个已经存在的线程挂了）或者进入这段函数时线程池关闭了；
>
> 3、如果不能入队，则创建一个新线程。如果失败，我们就知道线程池已经被关闭或已经饱和就需要调用拒绝策略来拒绝当前任务。

读完注释，哪怕我们不读代码或者读不懂源码，我们对线程池的理解也会较为深入的理解，读完注释后再读代码就会发现容易了很多。

我们再学习 `java.util.concurrent.ThreadPoolExecutor#shutdown` 函数：

```java
/**
     * Initiates an orderly shutdown in which previously submitted
     * tasks are executed, but no new tasks will be accepted.
     * Invocation has no additional effect if already shut down.
     *
     * <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```

根据注释我们可知：

> 已经提交的任务执行完后关闭，此时不会不再接收新的任务。
>
> 如果已经关闭，那么调用此函数没啥副作用。
>
> 此函数不会等待已经提交的任务执行完成（才返回）。如果需要可以使用 `java.util.concurrent.ThreadPoolExecutor#awaitTermination`。

假如我们对这里关键的一句话：“This method does not wait for previously submitted tasks to complete execution.” 很困惑，可以通过 StackOverFlow 搜索相关关键词来寻找解答。

我们找到这样一篇：[does-executorservice-shutdown-cancel-existing-tasks](https://stackoverflow.com/questions/9905010/does-executorservice-shutdown-cancel-existing-tasks) 文章

> The point is that the `shutDown` method *returns* without waiting for the previously submitted tasks to complete, but it still *lets* them complete. You might want to think of it as a “start shutting down” method.
>
> `shutDown` 不会等待直接提交的任务执行完成（但是会让它们执行完毕）就会返回。你可以将该方法理解为 “开始关闭” 函数。

线程池还有其它核心函数，需要大家自己去学习，这里就不作展开。

上面讲述了线程池的核心参数和核心函数。

那么我们来看另外一个问题，**为啥《手册》不建议用 `Executors` 来创建线程池？**

我们以 `FixedThreadPool` 为例，来分析具体原因：

```java
/**
 * Creates a thread pool that reuses a fixed number of threads
 * operating off a shared unbounded queue, using the provided
 * ThreadFactory to create new threads when needed.  At any point,
 * at most {@code nThreads} threads will be active processing
 * tasks.  If additional tasks are submitted when all threads are
 * active, they will wait in the queue until a thread is
 * available.  If any thread terminates due to a failure during
 * execution prior to shutdown, a new one will take its place if
 * needed to execute subsequent tasks.  The threads in the pool will
 * exist until it is explicitly {@link ExecutorService#shutdown
 * shutdown}.
 *
 * @param nThreads the number of threads in the pool
 * @param threadFactory the factory to use when creating new threads
 * @return the newly created thread pool
 * @throws NullPointerException if threadFactory is null
 * @throws IllegalArgumentException if {@code nThreads <= 0}
 */
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

通过工作上面的学习我们知道，工作队列是用来存放线程执行前的任务。

通过上面源码我们可以看出 `FixedThreadPool` 的核心线程数和最大线程数相等，而工作队列为

`java.util.concurrent.LinkedBlockingQueue`。

通过其默认构造方法，我们可以看出其容量为整数的最大值。

```java
/**
 * Creates a {@code LinkedBlockingQueue} with a capacity of
 * {@link Integer#MAX_VALUE}.
 */
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

根据前面学到的知识，我们试想一下这样的场景：

> 如果对该线程池的请求不断增多，达到核心线程数后，任务将暂存到该工作队列。但是这个阻塞队列是 “无界” 的，如果大量任务过来，工作队列可能还没达到整数最大值可能就已经 OOM 了。

如果我们自定义线程池对象，可以设置相对可控的最大线程数和可控的工作队列长度以及拒绝策略。那么即使任务大量堆积，在 OOM 之前就进入了拒绝策略。

总之通过自定义线程池参数，线程池的可控性更强。



###### 2.3.2 写 DEMO 大法

前面讲到 `java.util.concurrent.ThreadPoolExecutor#shutdown` 的功能，那么如何验证该函数的效果呢？

我们可以通过下面的例子来学习：

```java
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

@Slf4j
public class ThreadPoolShutDownDemo {

    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor executorService = new ThreadPoolExecutor(10, 10,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(50000), new NamedThreadFactory("shutdown-demo"));

        int total = 20000;
        for (int i = 0; i < total; i++) {
            executorService.submit(() -> {
                try {
                    TimeUnit.MILLISECONDS.sleep(5L);
                } catch (InterruptedException ignore) {
                }
                //System.out.println(Thread.currentThread().getName());
            });
        }
      
      // 第 1 处代码
        //executorService.shutdownNow();
        printExecutorInfo(total, executorService);
      
      // 第 2 处代码
        executorService.shutdown();
      
       // 第 3 处代码
        /* shutdown()之后再提交任务
        executorService.submit(() -> {
        });*/
      
       // 线程池没结束，隔一秒打印任务情况
        while (!executorService.isTerminated()) {
            TimeUnit.SECONDS.sleep(1);
            printExecutorInfo(total, executorService);
        }
    }

    /**
     * 打印线程池信息
     */
    private static void printExecutorInfo(int total, ThreadPoolExecutor executorService) {
        String dateString = DateUtil.toDateString(LocalDateTime.now(ZoneId.systemDefault()));
        log.debug("时间:{},总任务数：{}, 工作队列中有:{}个任务，已完成:{}个任务，正在执行:{}个任务", dateString, total, executorService.getQueue().size(), executorService.getCompletedTaskCount(), executorService.getActiveCount());
    }
}
```

执行效果如下：

> 时间：2019-08-24 20:58:50，总任务数：20000，工作队列中有：19900 个任务，已完成：90 个任务，正在执行：10 个任务
>
> …
>
> 时间：2019-08-24 20:59:02，总任务数：20000，工作队列中有：0 个任务，已完成：20000 个任务，正在执行：0 个任务

线程池没结束，每隔一秒打印一次线程池的任务信息。

从此示例中可以清楚地观察到调用 `executorService.shutdown()` 后 ，已经提交的任务仍然会被执行。

大家可以打开第 1 处代码，观察执行 `ThreadPoolExecutor#executorService.shutdownNow` 后如果继续提交任务将抛出 `RejectedExecutionException` 。

如果需要学习其他特性，大家都可以写一些简单的 DEMO，也可以断点调试观察更多细节。



#### 3. 总结

本节，我们再次使用源码法、StackOverFlow 大法、写 DEMO 法来学习线程池的一些知识点，包括线程池的核心参数，线程池的核心函数的源码和用法。

当然，大家还可以尝试断点调试法来进入核心函数来学习执行流程等。

下一节我们将带着大家深入研究：为何 JUnit 单元测试 “不支持多线程” 的问题。



#### 课后练习

1. 请大家根据本节学的内容，分析为什么不推荐使用 CachedThreadPool 的原因；
2. 通过本节课的学习，通过读源码和写 DEMO 的方式研究 `ThreadPoolExecutor` 的 `shutdownNow` 和 `shutdown` 函数的区别。



### 虚拟机退出时机问题研究



#### 1. 前言

前一节我们讲述了如何通过读源码，查询 StackOverFlow，写 DEMO 方式学习线程池。

然而线程池在使用过程中会遇到很多问题，本节将通过几个案例研究 Java 虚拟机关闭的问题。



#### 2. 背景知识

本节重点学习 JVM 关闭时机相关问题，那么 JVM 在何时正常退出呢（不包含通过 kill 指令杀死进程等情况）？

根据《 Java 虚拟机规范 (Java SE 8 版)》 第 228 页，对应英文版为 [5.7 Java Virtual Machine Exit](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.7) 的相关描述我们可知：

> Java 虚拟机退出的条件是，某个线程调用了 `Runtime` 类或 `System` 类的 `exit` 方法，或 `Runtime` 类的 `halt` 方法，并且 Java 安全管理器也允许这次 `exit` 或 `halt` 操作。
>
> 除此之外， JNI (Java Native Interface) 规范描述了用 JNI Invocation API 来加载或卸载 Java 虚拟机时，Java 虚拟机的退出情况 [1](https://www.imooc.com/read/55/article/1153#fn1)。

根据《Java 并发编程实践》 164 页相关论述 ，我们还了解到：

> 也可以通过一些其他平台相关的手段（比如发送 SIGINT, 或键入 Ctrl-C）, 都可以实现 JVM 的正常关闭。还可以调用 “杀死” JVM 的操作系统进程而强制关闭 JVM [2](https://www.imooc.com/read/55/article/1153#fn2)。

另外根据《Java Language Specification : Java SE 8 Edition》[12.8 Program Exit](https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.8) 的相关描述 [3](https://www.imooc.com/read/55/article/1153#fn3) 我们可知：

> 当下面两种情况发生时，程序将会结束所有活动并退出：
>
> - 只剩下守护线程（ daemon thread）时。
> - 某个线程调用了 `Runtime` 类或 `System 类` 的 `exit` 方法，并且 Java 安全管理器也允许这次 `exit` 操作。

了解这个背景知识，接下来我们将开始分析相关的案例。



#### 3. 案例及其分析



##### 3.1 JUnit 单元测试不支持多线程问题

本案例涉及两个类，一个是自定义线程类，一个是测试类。

自定义线程类：

```java
import java.util.concurrent.TimeUnit;

public class DemoThread extends Thread {


    public DemoThread() {
    }

    @Override
    public void run() {
        for (int i = 0; i < 4; i++) {
            System.out.println(Thread.currentThread().getName() + "-->" + i);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException ignore) {
            }
        }
    }

}
```

对应的单元测试：

```java
public class ThreadDemoTest {
    @Test
    public void test() throws InterruptedException {
        DemoThread demoThread1 = new DemoThread();
        DemoThread demoThread2 = new DemoThread();

        demoThread1.start();
        demoThread2.start();
    }
}
```

预期结果为，每个线程分别执行 4 次打印语句。

但是实际运行结果为：

> Thread-0–>0
> Thread-1–>0

打印两行文字后程序退出。

通过观察现象，我们看出 JUnit 单元测试 “不支持多线程” 测试，换句话说两个线程可能还没没执行完，程序就退出了。

我们首先尝试使用 **断点调试大法** 来寻找线索。
![图片描述](img/5dd4ddfa0001800f21361738.png)
我们通过查看左侧的调用栈，可以清晰地看到顶层的为 `com.intellij.rt.execution.junit.JUnitStarter#main` 的 70 行，通过一系列的调用，启动当前测试方法。

按照惯例，我们可以双击左侧的调用进入源码。

但是，**令人吐血的是，双击没反应**，崩溃中…

既然 IDEA 可以使用该类，那么显然此类可以被 IDEA 加载，根据最外层的入口包名（com.intellij.rt.execution.junit），我们断定不是 JDK 中的类，也不是我们 pom.xml 中引入的 jar 包中的类，应该是 idea 自己的类库。

我们去 IDEA 的安装目录去寻找线索。排查了 lib 文件夹下的所有 jar 包，发现和名称相匹配的 jar 包。
![图片描述](img/5dd4dde70001ae7207970213.png)

我们如何查看这几个 jar 中有没有源码和上面的匹配呢？

可以使用前面介绍的 Java 反编译工具： [JD-GUI](http://java-decompiler.github.io/)，查看这些包的源码。

由于我们使用的是 JUnit4 我们首先查看 junit-rt.jar 的反编译代码。
![图片描述](img/5dd4ddd800015dfd10220621.png)

我们在此处找到了 IDEA 调试时顶层的类！

从此反编译的代码可以看到， `main` 函数的 70 行。

```java
 int exitCode = prepareStreamsAndStart(array, agentName, listeners, name[0]);
```

该函数调用准备流和开始函数，并获得返回值作为退出码，然后调用 `System.exit(exitCode);` 退出 JVM。

因此问题就迎刃而解了。

我们重新梳理执行流程：

IDEA 运行 JUnit 4 时，

1. 先执行 `com.intellij.rt.execution.junit.JUnitStarter#main` ，此函数中调用 `prepareStreamsAndStart` 子函数；
2. 子函数最终调用到 `ThreadDemoTest#test` 的代码。
3. `ThreadDemoTest#test` 创建两个新线程并依次开启后结束，函数返回退出码，最终调用 `System.exit(exitCode);` 退出 JVM。

**那么如何避免两个子线程尚未执行完单元测试函数，就被主线程调用 `System.exit` 导致 JVM 退出呢？**

**方案 1：可以将代码写在 main 函数中；**

还记得开头说的吗，只要有一个非守护线程还在运行，虚拟机就不会退出（正常情况下）。

使用 main 函数代码非常简单，这里就不再提供。

**方案 2：可以使用 CountDownLatch；**

改造自定义的线程类：

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class DemoThread extends Thread {

    private CountDownLatch countDownLatch;

    public DemoThread(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        for (int i = 0; i < 4; i++) {
            System.out.println(Thread.currentThread().getName() + "-->" + i);
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException ignore) {
            }
        }
        countDownLatch.countDown();
    }

}
```

修改单元测试函数：

```java
@Test
public void test() throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(2);
    DemoThread demoThread1 = new DemoThread(countDownLatch);
    DemoThread demoThread2 = new DemoThread(countDownLatch);

    demoThread1.start();
    demoThread2.start();
    
    countDownLatch.await();
}
```

由于使用了 `countDownLatch.await();` 主线程会阻塞到两个线程都执行完毕。

具体原理大家可以查看 `java.util.concurrent.CountDownLatch#await()` 源码。

**方案 3：可以在测试函数最后调用 `join` 函数：**

```java
@Test
public void test() throws InterruptedException {
    DemoThread demoThread1 = new DemoThread();
    DemoThread demoThread2 = new DemoThread();

    demoThread1.start();
    demoThread2.start();

    demoThread1.join();
    demoThread2.join();
}
```

join 函数会等待当前线程执行结束再继续执行。



##### 3.2 使用 CompletableFuture 的问题

大家可以猜想一下下面代码的执行结果是啥？

```java
public class CompletableFutureDemo {
    
    public static void main(String[] args) {
        CompletableFuture.runAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(2L);
            } catch (InterruptedException ignore) {
            }
            System.out.println("异步任务");
        });
    }
}
```

可能出乎很多人的意料，如果运行此段代码，大概率会发现：打印语句并没有被执行程序就退出了。

What? ** 前面不是说多线程问题可以通过将代码写在 main 函数中来避免的吗？** 怎么瞬间打脸？

别急，我们来研究一下这个问题：

```java
/**
 * Returns a new CompletableFuture that is asynchronously completed
 * by a task running in the given executor after it runs the given
 * action.
 *
 * @param runnable the action to run before completing the
 * returned CompletableFuture
 * @param executor the executor to use for asynchronous execution
 * @return the new CompletableFuture
 */
public static CompletableFuture<Void> runAsync(Runnable runnable,
                                               Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```

通过源码注释，我们可知该函数是使用给定的 `executor` 来异步执行任务。

那么使用的线程池类型是什么呢？

```java
/**
 * Null-checks user executor argument, and translates uses of
 * commonPool to asyncPool in case parallelism disabled.
 */
static Executor screenExecutor(Executor e) {
    if (!useCommonPool && e == ForkJoinPool.commonPool())
        return asyncPool;
    if (e == null) throw new NullPointerException();
    return e;
}
```

我们查看 `asyncPool` 的具体类型：

```java
/**
   * Default executor -- ForkJoinPool.commonPool() unless it cannot
   * support parallelism.
   */
  private static final Executor asyncPool = useCommonPool ?
      ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();

  /** Fallback if ForkJoinPool.commonPool() cannot support parallelism */
  static final class ThreadPerTaskExecutor implements Executor {
     public void execute(Runnable r) { new Thread(r).start(); }
   }
```

默认是 `ForkJoinPool.commonPool()` ，如果不支持并行则会构造一个新的 `ThreadPerTaskExecutor` 线程池对象。

我们再次回到正题，我们可以查看调用链：

```
java.util.concurrent.CompletableFuture#runAsync(java.lang.Runnable)
java.util.concurrent.CompletableFuture#asyncRunStage
java.util.concurrent.ForkJoinPool#execute(java.lang.Runnable)
java.util.concurrent.ForkJoinPool#externalPush
```

…

最终调用到：

```
java.util.concurrent.ForkJoinPool#registerWorker
```

如下图所示，大家可以在 `registerWorker` 函数的设置守护线程代码的地方打断点，然后调试，通过查看左侧 “Debugger” 选项卡的 “Frames” 调用栈来研究整个调用过程，也可以切换到 “Threads” 来查看线程的运行状态。
![图片描述](img/5dd4ddc200016f8e13170672.png)

接下来我们看源码：

```java
/**
 * Callback from ForkJoinWorkerThread constructor to establish and
 * record its WorkQueue.
 *
 * @param wt the worker thread
 * @return the worker's queue
 */
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    // 第 1 处
    wt.setDaemon(true);                           // configure thread
    // 省略中间代码
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    return w;
}
```

从这里可知 `ForkJoinPool` 的工作线程类型为守护者线程。

根据前面背景知识的介绍，我们可知如果只有守护线程，程序将退出。

另外，我们也可以从设置守护线程的函数中找到相关描述：

```java
/**
 * Marks this thread as either a {@linkplain #isDaemon daemon} thread
 * or a user thread. The Java Virtual Machine exits when the only
 * threads running are all daemon threads.
 *
 * <p> This method must be invoked before the thread is started.
 *
 * @param  on
 *         if {@code true}, marks this thread as a daemon thread
 *
 * @throws  IllegalThreadStateException
 *          if this thread is {@linkplain #isAlive alive}
 *
 * @throws  SecurityException
 *          if {@link #checkAccess} determines that the current
 *          thread cannot modify this thread
 */
public final void setDaemon(boolean on) {
    checkAccess();
    if (isAlive()) {
        throw new IllegalThreadStateException();
    }
    daemon = on;
}
```

因此我们重新分析上面的案例：

```java
public static void main(String[] args) {
  // 第 1 处
    CompletableFuture.runAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2L);
        } catch (InterruptedException ignore) {
        }
        System.out.println("异步任务");
    });
  // 第 2 处
}
```

主线程为普通用户线程，执行到第 1 处，使用默认的 `ForkJoinPool` 来异步执行传入的任务。

此时工作线程（守护线程）如果得到运行机会，调用 `TimeUnit.SECONDS.sleep(2L);` 导致该线程 `sleep` 2 秒钟。

主线程执行到第 2 处 （无代码），然后主线程执行完毕。

此时已经没有非守护线程，还不等工作线程从 Time waiting 睡眠状态结束，虚拟机发现已经没有非守护线程，便退出了。



##### 3.3 拓展练习

有了上面的介绍，想必大家对虚拟机的退出时机有了一个不错的了解，那么我们看下面的代码片段：

**请问程序执行后是否一定执行到 finally 代码块，为什么？**

```java
public class Demo {

    public static void main(String[] args) {
       // 省略一些代码 （第 1 处）
        try {
            BufferedReader br = new BufferedReader(new FileReader("file.txt"));
            System.out.println(br.readLine());
            br.close();
        } catch (Exception e) {
            // 省略一些代码 （第 2 处）
        } finally {
            System.out.println("Exiting the program");
        }
    }
}
```

结合今天所学内容，很多朋友可能会想到，在第 2 处如果让当前虚拟机退出，那么 finally 代码块就不会再执行。

因此可以添加 `System.exit(2)` 来实现。

当然还有其他的方法能够实现，大家可以在评论区畅所欲言。



#### 4. 总结

本节重点讲述了虚拟机退出的条件，举了几个案例让大家能够对此有深刻的理解。

本节使用了读源码法，官方文档法，断点调试法等来分析这两个案例。

下一节我们将讲述如何解决多条件语句和条件语句的多层嵌套问题。



#### 5. 思考题

请看下面代码片段，回答问题。

```java
public class Demo {
 
    public static void main(String[] args) {
 
      // 省略一些代码 （第 1 处）
        try {
            BufferedReader br = new BufferedReader(new FileReader("file.txt"));
            System.out.println(br.readLine());
            br.close();
        } catch (Exception e) {
           System.exit(2);
        } finally {
            System.out.println("Exiting the program");
        }
    }
}
```

问题：**如果 try 代码块发生异常，如何在第 1 处代码添加几行代码，使得 finally 代码块可以被执行到呢？**



### 如何解决条件语句的多层嵌套问题？



#### 1. 前言

《手册》第 19 页，有关于多 if-else 分支和嵌套的建议和解决方案 [1](https://www.imooc.com/read/55/article/1154#fn1)：

> 表达分支时，如果非要使用 if ()…else if ()…else… 方式表达逻辑，避免后续代码维护困难，不允许超过三层。
>
> 如果超过 3 层可以使用卫语句、策略模式、状态模式等来实现。
>
> 其中卫语句代码逻辑优先考虑失败、异常、中断、退出等直接返回的情况。

那么我们要思考以下几个问题：

- 我们该如何将这几种方案落地呢？
- 使用过程中会遇到哪些奇葩的问题呢？

这些都是本节重点研究的问题。

请看下面开发中可能会遇到的典型代码：

```java
  public double getSalary(Integer position) {
       double result;
       if (position == null) {
          throw new IllegalArgumentException("职位不能为空");
       }

       // 老板
       if (isBoss(position)) {
           result = getBossSalary();
       } else {
          // 领导
          if (isLeader(position)) {
               result = getLeaderSalary();
          } else {
               // 普通员工
              result = getStaffSalary();
          }
      }
      return result;
}
```

我们如何替代多分支和分支嵌套问题呢？如何让代码变得更容易维护和拓展呢？

请看下面的分析。



#### 2. 卫语句

《重构》 第 9 章 9.5 节 以卫语句取代嵌套条件表达式 中，有如下描述：

> 如果某个条件极其罕见，就应该单独检查该条件，并在条件为真时立即从函数中返回。这样的单独检查常常被称为 “卫语句”。
>
> 卫语句要不就从函数中返回，要不就抛出一个异常。

使用卫语句，我们可以对上面的示例修改为：

```java
public double getSalaryGuard(Integer position) {

    // 条件检查
    if (position == null) {
        throw new IllegalArgumentException("职位不能为空");
    }
    // 老板
    if (isBoss(position)) {
        return getBossSalary();
    }
    // 领导
    if (isLeader(position)) {
        return getLeaderSalary();
    }
    // 普通员工
    return getStaffSalary();
}
```

先进行条件检查，然后将 if-else 逻辑转成对应的卫语句格式。

另外我们还可以参考 `org.apache.commons.lang3.ObjectUtils#isEmpty` 的源码：

```java
public static boolean isEmpty(final Object object) {
       // 第 1 处
        if (object == null) {
            return true;
        }
        // 第 2 处
        if (object instanceof CharSequence) {
            return ((CharSequence) object).length() == 0;
        }
        // 第 3 处
        if (object.getClass().isArray()) {
            return Array.getLength(object) == 0;
        }
  
       // 第 4 处
        if (object instanceof Collection<?>) {
            return ((Collection<?>) object).isEmpty();
        }
         // 第 5 处
        if (object instanceof Map<?, ?>) {
            return ((Map<?, ?>) object).isEmpty();
        }
        return false;
    }
```

第 1 处代码满足：某个条件极其罕见，就应该单独检查该条件，并在条件为真时立即从函数中返回。

第 2 到第 5 处代码将某个分支条件转化成卫语句。

在这里特别提醒的是：**对于复杂的判断逻辑，选择使用卫语句时，建议加上注释，并且要仔细核实逻辑是否正确**。

请看下面的伪代码：

```java
// 第 1 处
// 同时满足 a 和 b 两个条件
if(condition_a && condition_b){
   if(conditon_c){
     // 业务代码
     return;
   }
}

// 第 2 处
// 条件a 和 b至少有一个不满足
if(!conditon_c){
  // 业务代码
  return;
}
```

上面代码看似正确，其实有很大的问题。

如果同时满足条件 a 和 条件 b 且不满足条件 c，代码依然会执行到 第 2 处，此时 “条件 a 和 b 同时满足” 和 第 2 处的的注释 “条件 a 和 b 至少有一个不满足” 不一致。

我们需要对代码做出如下修改：

```java
// 第 1 处
// 同时满足 a 和 b 两个条件
if(condition_a && condition_b){
   if(conditon_c){
     // 业务代码
   }
// 第 3 处
   // 保证整个if执行后返回
   return;
}

// 第 2 处
// 条件a 和 b至少有一个不满足
if(!conditon_c){
  // 业务代码
  return;
}
```

因此使用卫语句是要特别注重卫语句的先后顺序，当条件非常复杂时，要特别注意卫语句的中断是否符合希望的逻辑。



#### 3. 策略枚举

正如前面的枚举小节讲到的，《Effective Java 中文版》 第 34 条 ：用 enum 代替 int 常量 [2](https://www.imooc.com/read/55/article/1154#fn2) 小节描述了使用策略枚举，来替代分支语句，虽然失去了简洁性，但是更加安全和灵活。

通过在枚举内部定义抽象函数，每个枚举常量重写该函数，这样根据枚举值获取枚举常量后调用该函数即可获得期待的计算结果。

示例代码如下：

```java
public enum SalaryStrategyEnum {

    BOSS(0) {
        @Override
        double getSalary() {
            return 100000;
        }
    },
    LEADER(1) {
        @Override
        double getSalary() {
            return 50000;
        }
    },
    STAFF(2) {
        @Override
        double getSalary() {
            return 10000;
        }
    };

    private final int position;

    SalaryStrategyEnum(int position) {
        this.position = position;
    }

    abstract double getSalary();

    public static SalaryStrategyEnum valueOf(int position) {
        for (SalaryStrategyEnum salaryStrategyEnum : SalaryStrategyEnum.values()) {
            if (salaryStrategyEnum.position == position) {
                return salaryStrategyEnum;
            }
        }
        return null;
    }
}
```

使用时根据枚举值获取枚举对象，直接调用该枚举常量对应的策略:

```java
@Test
public void getSalary() {
    SalaryStrategyEnum salaryStrategyEnum = SalaryStrategyEnum.valueOf(0);
    if(salaryStrategyEnum != null){
        log.info("角色：{}-->{} 元",salaryStrategyEnum.name(),salaryStrategyEnum.getSalary());
    }
}
```

当然，大家也可以用非枚举的策略模式来替代多个条件语句。

> 看到这里，可能有些人会认为这种写法工作中并不会用到。
>
> 实则不然，很多知识是你真正理解之后就会想到使用它，恰恰是自认为没用和没有真正理解才导致工作不能灵活运用。
>
> 在工作中，看到多个项目涉及到根据不同枚举计算不同的值时，都用到过类似的写法。



#### 4. 状态模式

《设计模式之禅》 第 26 章 状态模式 (第 343 页) 中讲到：

> 状态模式的使用场景有两类：一种是行为随着状态改变而改变的场景；另外一种是条件、分支判断语句的替代者。

状态模式的其中一个优点就是 “结构清晰”。状态模式体现了开闭原则和单一职责原则，易于拓展和维护。

所谓的结构清晰就是避免了过多的 switch-case 或者 if-else 语句的使用，避免了程序的复杂性，提高了程序的可维护性 [3](https://www.imooc.com/read/55/article/1154#fn3)。

接下来我们采用状态模式通过另外一个例子来演示。

原始的 if-else 语句和文章首部给出的非常类似，根据当前状态来执行不同的行为：

学生类：

```java
@Getter
@Setter
public class Student {

    private Long id;

    private String name;

    private Long age;
}
```

对应的根据状态执行不同的处理函数代码：

```java
private void doAction(Integer state, Student student) {

    if (state == null) {
        throw new IllegalArgumentException("状态不能为空");
    }

    switch (state) {
        case 0:
            enroll(student);
            break;
        case 1:
            study(student);
            break;

        case 2:
            graduate(student);
            break;
        default:
    }
}

/**
 * 入学
 */
private void enroll(Student student) {
    System.out.println(String.format("学生%s报名中....", student.getName()));
}

/**
 * 学习
 */
private void study(Student student) {
    System.out.println(String.format("学生%s正在学习....", student.getName()));
}

/**
 * 毕业
 */
private void graduate(Student student) {
    System.out.println(String.format("学生%s毕业了....", student.getName()));
}
```

接下来我们使用状态模式对上面的示例进行修改。

对应的类图如下：
![图片描述](img/5ddb314700013dec06301562.png)

State 接口或者抽象类负责对象状态的定义。

Context 定义客户端所需的接口，并且负责状态的切换。

状态抽象类：

```java
@Data
public abstract class State {
    protected Context context;

    protected State nextState;

    public void setContext(Context context) {
        this.context = context;
    }

    abstract void doAction();
}
```

报名状态：

```java
/**
 * 报名状态
 */
public class EnrollState extends State {

    public EnrollState() {
        super();
        nextState = new StudyState();
    }

    @Override
    public void doAction() {
        System.out.println(String.format("学生%s报名中....", context.getStudent().getName()));
    }
}
```

学习状态：

```java
/**
 * 学习状态
 */
public class StudyState extends State {

    public StudyState() {
        nextState = new GraduateState();
    }

    @Override
    public void doAction() {
        System.out.println(String.format("学生%s正在学习....", context.getStudent().getName()));
    }
}
```

毕业状态：

```java
/**
 * 毕业状态
 */
public class GraduateState extends State {

    public GraduateState() {
        nextState = null;
    }

    @Override
    public void doAction() {
        System.out.println(String.format("学生%s毕业了....", context.getStudent().getName()));
    }
}
```

上下文类：

```java
public class Context {
    private Student student;
    private State currentState;

    public void doAction() {
        currentState.doAction();
    }
    
    public State getCurrentState() {
        return currentState;
    }

    public void setCurrentState(State currentState) {
        this.currentState = currentState;
        this.currentState.setContext(this);
    }

    public State getNextSate() {
        return currentState.nextState;
    }

    public Student getStudent() {
        return student;
    }

    public void setStudent(Student student) {
        this.student = student;
    }
}
```

具体使用：

```java
public class StateClinet {
    public static void main(String[] args) {
        Student student = new Student();
        student.setName("tomcat");

        Context context = new Context();
        context.setStudent(student);

        // 报名状态
        context.setCurrentState(new EnrollState());
        context.doAction();

        // 学习状态
        State nextSate = context.getNextSate();
        while (nextSate != null) {
            context.setCurrentState(nextSate);
            nextSate.doAction();
            nextSate = nextSate.nextState;
        }
    }
}
```

输出：

> 学生 tomcat 报名中…
> 学生 tomcat 正在学习…
> 学生 tomcat 毕业了…

上述示例通过状态模式解决了条件嵌套问题。



#### 5. 拦截器过滤器模式

如果是 Spring Web 项目中还可以通过实现 `org.springframework.context.ApplicationContextAware` 接口，构造待处理的类型到对应处理器的映射，这也是简化 if-else if-else 的一个重要手段，在实际开发中这种方式也很常见。

定义校验基类：

```java
@Data
public abstract class Validator<P> {
    /**
     * 校验分组，枚举
     */
    private Set<Enum> groups;

    /**
     * 验证参数
     */
    abstract void validate(P param);
}
```

自定义校验器：

```java
@Component
public class UserSexValidator extends Validator<UserParam> {

    @Override
    void validate(UserParam param) {
        System.out.println("验证性别");
        if (param == null) {
            throw new BusinessException("");
        }
        // 模拟服务，根据userId查询性别
        boolean isFemale = RandomUtils.nextBoolean();
        if (!isFemale) {
            throw new BusinessException("仅限女性玩家哦！");
        }
    }
}
```

通过继承上述父类，可以自定义针对某个类的各种类型的校验器。

构造校验类和校验处理器的映射：

```java
@Component
public class ValidatorChain implements ApplicationContextAware {

    private Map<Class, List<Validator>> validatorMap = new HashMap<>();

    /**
     * 根据自定义的校验器进行参数校验
     */
    public <P> void checkParam(P param) {
        checkParam(param, validator -> true);
    }

    /**
     * 符合某种条件才参数校验
     */
    public <P> void checkParam(P param, Predicate<Validator> predicate) {
        List<Validator> validators = getValidators(param.getClass());
        if (CollectionUtils.isNotEmpty(validators)) {
            validators.stream()
                    .filter(predicate)
                    .forEach(validator -> validator.validate(param));
        }
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        Map<String, Validator> beansOfType = applicationContext.getBeansOfType(Validator.class);
        this.validatorMap = beansOfType.values().stream().collect(Collectors.groupingBy(validator -> getParamType(validator.getClass())));
    }

    /**
     * 查找相关的所有校验器
     */
    private List<Validator> getValidators(Class clazz) {
        return validatorMap.get(clazz);
    }

    /**
     * 解析泛型待校验参数类型
     */
    private Class getParamType(Class clazz) {
        ResolvableType resolvableType = ResolvableType.forClass(clazz);
        return resolvableType.getSuperType().getGeneric(0).resolve();
    }
}
```

使用校验链：

```java
@Service
public class UserServiceImpl implements UserService {

    @Resource
    private ValidatorChain validatorChain;

    @Override
    public UserDTO checkUser(UserParam userParam) {

        // 参数校验
        validatorChain.checkParam(userParam);
        // 业务逻辑
        return new UserDTO("测试");
    }

    @Override
    public UserDTO checkUserSome(UserParam userParam) {
        // 参数校验(只校验类型为Some的)
        validatorChain.checkParam(userParam, param -> param.getGroups().contains(UserValidateGroupEnum.SOME));

        // 业务逻辑
        return new UserDTO("测试");
    }
}
```

通过这种方式，对于不同类型对象的属性校验不需要通过 if -else 判断，新增某种类型的校验只需要添加一个自定义校验器即可。还可以支持通过 lambda 表达式传入过滤条件，让符合条件的自定义校验器生效。



#### 6. 嵌套条件语句

嵌套条件语句是多条件语句的变种，相当于增加了内层的一个或者多个嵌套层次。

实际开发中可以将多次使用同一个设计模式，也可以将各种设计模式综合在一起使用。

下面以一个简单的具体例子为例，为大家讲解如何解决嵌套的条件的情况：

用户类：

```java
import lombok.Data;

@Data
public class User {

    private Short age;

    private Boolean male;

    private Long id;

    private String name;
}
```

示例代码：

```java
public class Demo {

    public static void main(String[] args) {
        Demo demo = new Demo();
        User user = new User();
        user.setAge((short) 17);
        user.setMale(true);
        demo.some(user);
    }

    private void some(User user) {
        Short age = user.getAge();
        if (age < 18) {
            if (user.getMale()) {
                // 其他代码
                System.out.println("18岁 以下男性");
            } else {
                // 其他代码
                System.out.println("18岁 以下女性");
            }
        } else if (age <= 60) {
            if ("张三".equals(user.getName())) {
                // 其他代码
                System.out.println("18到60岁 张三");
            } else if ("李四".equals(user.getName())) {
                System.out.println("18到60岁 李四");
            } else {
                // 其他代码
                System.out.println("18到60岁 其他");
            }
        } else {
            System.out.println("60岁以上");
        }
    }
}
```

接下来，我们使用职责链模式和 Map （也可以用标准的工厂模式）对该条件嵌套示例进行重构。

抽象类：

```java
import java.util.function.Predicate;

public abstract class AbstractAgeHandler {

    /**
     * 下一个处理器
     */
    protected AbstractAgeHandler nextAgeHandler;

    /**
     * 设置下一个处理器
     */
    public void setNextAgeHandler(AbstractAgeHandler nextAgeHandler) {
        this.nextAgeHandler = nextAgeHandler;
    }

    public void handle(User user) {
        if (getCondition().test(user.getAge())) {
            doHandle(user);
        }
        if (nextAgeHandler != null) {
            nextAgeHandler.handle(user);
        }
    }

    /**
     * 实际处理函数
     */
    protected abstract void doHandle(User user);

    /**
     * 获取查询条件
     */
    public abstract Predicate<Short> getCondition();
}
```

小于 18 岁的处理：

```java
mport java.util.HashMap;
import java.util.Map;
import java.util.function.Predicate;

public class LessThan18Handler extends AbstractAgeHandler {

    // 存储策略
    private static final Map<Boolean, Runnable> SEX_STRATEGY_MAP = new HashMap<>();

    static {
        SEX_STRATEGY_MAP.put(Boolean.TRUE, () -> {
            // 一种处理策略
            System.out.println("小于18岁  男性");
        });

        SEX_STRATEGY_MAP.put(Boolean.FALSE, () -> {
            // 另外的处理策略
            System.out.println("小于18岁  女性");
        });

    }

    /**
     * 该条件的处理函数
     */
    @Override
    protected void doHandle(User user) {
        // 处理小于18岁的代码逻辑

        // 处理性别部分
        SEX_STRATEGY_MAP.get(user.getMale()).run();
    }

    /**
     * 条件18岁
     */
    @Override
    public Predicate<Short> getCondition() {
        return (age) -> age < 18;
    }
}
```

18 到 60 岁之间的处理：

```java
import java.util.HashMap;
import java.util.Map;
import java.util.function.Predicate;

public class Between18And60Handler extends AbstractAgeHandler {
    // 存储策略
    private static final Map<String, Runnable> NAME_STRATEGY_MAP = new HashMap<>();

    static {
        NAME_STRATEGY_MAP.put("张三", () -> {
            // 一种处理策略
            System.out.println("18到60岁 张三");
        });

        NAME_STRATEGY_MAP.put("李四", () -> {
            // 另外的处理策略
            System.out.println("18到60岁 李四");
        });

    }

    /**
     * 该条件的处理函数
     */
    @Override
    protected void doHandle(User user) {
        // 处理18 到60岁的代码逻辑

        // 处理性别部分
        Runnable runnable = NAME_STRATEGY_MAP.get(user.getName());
        if (runnable != null) {
            runnable.run();
        } else {
            System.out.println("18到60岁 其他");
        }
    }

    /**
     * 条件18岁
     */
    @Override
    public Predicate<Short> getCondition() {
        return (age) -> age >= 18 && age <= 60;
    }
}
```

大于 60 岁的处理方式：

```java
import java.util.function.Predicate;

public class MoreThan60Handler extends AbstractAgeHandler {


    @Override
    protected void doHandle(User user) {
        System.out.println("没有分支逻辑，支持处理");
    }

    @Override
    public Predicate<Short> getCondition() {
        return (age) -> age > 60;
    }
}
```

示例代码：

```java
public class Demo {
    public static void main(String[] args) {

        // 构造年龄处理器
        AbstractAgeHandler first = new LessThan18Handler();
        AbstractAgeHandler second = new Between18And60Handler();
        AbstractAgeHandler third = new MoreThan60Handler();

        // 编排
        first.setNextAgeHandler(second);
        second.setNextAgeHandler(third);

        // 使用
        User user = new User();
        user.setAge((short) 19);
        user.setMale(true);

        first.handle(user);
    }
}
```

究竟选择哪种设计模式要结合具体的场景。

大家可以通过《设计模式之禅》、《Head Ffirst 设计模式》和菜鸟教程等学习常见的设计模式，了解其适合的场景，优缺点等，根据具体场景灵活使用。



#### 7. 总结

本节主要讲了如何解决 if-else 语句拓展性和多层嵌套问题。可以通过卫语句、策略模式、状态模式和过滤拦截器模式等方式解决。

希望大家能够在实际开发中尝试使用这些方法来编写更加优雅的代码。

下一节我们将学习异常处理的相关知识，给出异常处理不当的坑，还会给出一些异常处理的建议。



### 加餐1：工欲善其事必先利其器



#### 1.前言

俗话说：“工欲善其事，必先利其器”。

为了助力大家的学习和进阶，本小节介绍几个对 Java 学习非常有帮助的 IDEA 插件，代码反编译和反汇编工具，以及非常不错的网站等。



#### 2. IDEA 插件

首先不必多说，IDEA 是目前 Java工程师最主流的开发工具， IDEA 的强大之处不仅在于自身，还在于提供了丰富的插件（这点和谷歌浏览器非常类似）。

本部分介绍几款强大实用的 IDEA 插件，助力大家开发。

以下插件大都可以通过 IDEA 自带的插件管理中心安装，如果搜不到可以去 IDEA 插件官网下载本地导入。
![图片描述](img/5ddb3f380001e5f212480751.png)

具体安装界面不同版本 IDEA略有差异，请自行研究。

如果连插件安装都不愿意学、学不会的话，很难成为一名合格的 Java 开发工程师。



##### 2.1 Alibaba Java Coding Guidelines

首先要推荐的是和《手册》配套的[阿里巴巴 Java代码规范插件](https://plugins.jetbrains.com/plugin/10046-alibaba-java-coding-guidelines)。

安装该插件后，代码超过 80 行、手动创建线程池等，这些和《手册》中的规约不符时，IDEA中会给出警告提示。

建议大家一定一定一定要安装该插件，它会帮助你检查出很多隐患，督促你写更规范的代码。



##### 2.2 jclasslib bytecode viewer

下面要隆重介绍的是一款可视化的字节码查看插件：[jclasslib](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer) 。

大家可以直接在 IDEA 插件管理中安装（安装步骤略）。

**使用方法**：

1. 在 IDEA 打开想研究的类；
2. 编译该类或者直接编译整个项目（ 如果想研究的类在 jar 包中，此步可略过）；
3. 打开“view” 菜单，选择“Show Bytecode With jclasslib” 选项；
4. 选择上述菜单项后 IDEA 中会弹出 jclasslib 工具窗口。

![图片描述](img/5ddb3f20000117a812180637.png)

那么有自带的强大的反汇编工具 javap 还有必要用这个插件吗？

这个插件的**强大之处**在于：

1. 不需要敲命令，简单直接，在右侧方便和源代码进行对比学习；
2. 字节码命令支持超链接，**点击其中的虚拟机指令即可跳转到 jvms 相关章节**，超级方便。

该插件对我们学习虚拟机指令有极大的帮助。



##### 2.3 Codota

另外一个不得不说的就是专栏中提到的辅助开发神器: [Codota](https://www.codota.com/code)。

可以点击下图所示“Add Codota to you IDEA” 了解安装步骤。

![图片描述](img/5ddb3f090001806121041382.png)
该插件的强大之处在于：

1. 支持智能代码自动提示，该功能可以增强 IDEA 的代码提示功能；
2. 支持 JDK 和知名第三方库的函数的使用方法搜索，可以看到其他知名开源项目对该函数的用法。

当我们第一次使用某个类，对某个函数不够熟悉时，可以通过该插件搜索相关用法，快速模仿学习。
![图片描述](img/5ddb3ef60001b12614920677.png)

如上图所示，我们想了解 `Stream` 类中 `flatMap` 函数的用法，可以使用该插件查看知名开源项目的用法。

插件窗口顶部还给出了该类最常用的函数，可以点击查看相关用法案例，每个案例右侧的 "view source"可以跳转到该片段对应的开源项目的源码中。



##### 2.4 Auto filling Java call arguments

开发中，我们通常会调用其它已经编写好的函数，调用后需要填充参数，但是绝大多数情况下，传入的变量名称和该函数的参数名一致，当参数较多时，手动单个填充参数非常浪费时间。

该插件就可以帮你解决这个问题。

安装完该插件以后，调用一个函数，使用 Alt+Enter 组合键，调出 “Auto fill call parameters” 自动使用该函数定义的参数名填充。



##### 2.5 GenerateO2O、**GenerateAllSetter**

我们定义好从 A 类转换到 B 类的函数转换函数后，使用这两个插件可以自动调用 Getter 和 Setter 函数实行自动转换。

实际开发中还有一个非常常见的场景： 我们创建一个对象后，想依次调用 Setter 函数对属性赋值，如果属性较多很容易遗漏或者重复。
![图片描述](img/5ddb3ed5000175bd05360232.png)

可以使用这 GenerateAllSetter 提供的功能，自动调用所有 Setter 函数（可填充默认值），然后自己再跟进实际需求设置属性值。



##### 2.6 Material Theme UI

对于很多人而言，写代码时略显枯燥的，如果能够安装自己喜欢的主题将为开发工作带来些许乐趣。

IDEA 支持各种主题插件，其中最出名的当属 Material Theme UI。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)
安装后，可以从该插件内置的各种风格个选择自己最喜欢的一种。



##### 2.7 Rainbow Brackets

由于很多人没有养成好的编码风格，没有随手 format 代码的习惯，甚至有些同事会写代码超过几百行，阅读起来将非常痛苦。

痛苦的原因之一就是找到上下文，由于括号太多，不确定当前代码行是否属于某个代码块，此时这个插件就会帮上大忙。

插件 github 地址：https://github.com/izhangzhihao/intellij-rainbow-brackets。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

大家可以观看其 github 首页的动图体会和学习其强大功能。



##### 2.8 Maven Helper

现在 Java 项目通常会使用 maven 或者 gradle 构建，对于maven 项目来说， jar 包冲突非常常见。

那么如何更容易地查看和解决 jar 包冲突呢？
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

大家可以安装该插件，安装后 IDEA 中打开 pom.xml 文件时，就会多出一个 “Dependency Analyzer” 选项卡。

如上图所示，该插件支持值插件冲突的 jar 包，可以选择冲突的 jar 包将其 exclude 掉。



##### 2.9 FindBugs

程序员总是想尽可能地避免写 BUG， [FindBugs](https://plugins.jetbrains.com/plugin/3847-findbugs-idea) 作为静态代码检查插件，可以检查你代码中的隐患，并给出原因。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

当然还有很多类似的静态代码检查插件，大家可以自行了解安装。



##### 2.10 SequenceDiagram

[SequenceDiagram ](https://plugins.jetbrains.com/plugin/8286-sequencediagram/)可以根据代码调用链路自动生成时序图，超级赞，超级推荐！

这对研究源码，梳理工作中的业务代码有极大的帮助，堪称神器。

安装完成后，在某个类的某个函数中，右键 --> Sequence Diagaram 即可调出。

如下图是 Netty 的源码，可以通过该插件绘制出当前函数的调用链路。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

双击顶部的类名可以跳转到对应类的源码中，双击调用的函数名可以直接调入某个函数的源码，总之非常强大。



##### 2.11 Stack trace to UML

[Stack trace to UML](https://plugins.jetbrains.com/plugin/10749-stack-trace-to-uml/) 支持根据 JVM 异常堆栈画 UML时序图和通信图。

打开方式 *Analyze > Open Stack trace to UML plugin* + Generate UML diagrams from stacktrace from debug

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 2.12 Java Stream Debugger

Stream 非常好用，可以灵活对数据进行操作，但是对很多刚接触的人来说，不好理解。

那么 [Java Stream Debugger](https://plugins.jetbrains.com/plugin/9696-java-stream-debugger/) 这款神器的 IDEA 就可以帮到你。它可以将 Stream 的操作步骤可视化，非常有助于我们的学习。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 2.13 其它

IDEA 的插件浩如烟海，好的IDEA 插件欢迎留言交流。

另外大家可以通过[ IDEA插件官网](https://plugins.jetbrains.com/)进行搜索，有海量插件供你选择。



#### 3.反编译和反汇编软件

Java 学习进阶之路离不开Java 反编译和反汇编。

实际开发中需要用到反汇编的典型场景有：

- 自己或者二方上传的包含新的接口 jar 包到maven 仓库，下载下来查看 jar 包检查新的接口是否包含在新的 jar 包中；
- 需要临时查看某个 Jar 包的源码，不想加到本地仓库中；
- 拿不到源码，又想了解其源码究竟是怎么写的；
- 线上代码表现和自己的源码不一致，怀疑线上代码不对，可以反编译去核对。

对于大多数普通 Java 工程师来说，使用反编译的场景多是为了学习研究。



##### 3.1 在线Java反编译工具

有很多在线反编译的网站，其中比较好用的主要是以下两个：

http://www.decompiler.com/
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

使用简单，直接将 jar 包和 class文件拖到页面即可。

http://www.javadecompilers.com/
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

功能很强大，支持多种反编译方式，但是浏览效果不如上面网站好。



##### 3.2 离线 Java 反编译工具



###### 3.2.1 反编译软件

很多人担心在线反编译可能会引起代码泄露等，所以倾向于使用本地的反编译工具。

这里推荐两款软件： JD-GUI 和 Luyten。

[JD-GUI](http://java-decompiler.github.io/) 是一款可以根据 Java的 class 文件反编译出其源码的工具，界面简单，功能强大。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

另外一个非常好用的反编译软件为 [Luyten](https://github.com/deathmarine/Luyten)， 它是反编译工具 Procyon 的可视化显示工具。

大家可以在其 github 上下载安装：https://github.com/deathmarine/Luyten/releases。

该软件的用法和 JD-GUI 类似。

**图形界面反编译虽然更直观，但是如果我们想反编译Linux服务器上的类文件可咋办呢？**

我们可以通过 [Jad](http://www.kpdus.com/jad.html#download) 、[CFR](http://www.benf.org/other/cfr/)、[Procyon](https://github.com/ststeiger/procyon)、ernflower、 JD等反编译工具。

另外知名的阿里开源 Java诊断工具 arthas 也支持 [jad 命令](https://alibaba.github.io/arthas/jad.html)，可以将 JVM 中实际运行的 class 文件的字节码反编译成 Java 代码，便于理解业务和排查问题。

> 举一个真实发生过的典型的场景：
>
> 有一次代码发布上线，但是从功能表现看线上仍然是“旧代码”，但是从发布的 git 提交版本来看是最新版。
>
> 此时就可以使用 jad 反编译该类，来核查该问题。



###### 3.2.2 反汇编

这里简单介绍 Java 反编译和反汇编的区别。

这里说的反编译是指：将 class 文件反编译成 Java 源码的过程。

这里说的反汇编是指：将class 文件反解析为更可读的虚拟机指令的过程。

反汇编最权威和强大的当属 JDK 自带的 javap 工具，具体用法直接输入帮助指令`javap -help` 即可查看：

```
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

大家一定要自己多动手实践，才能更好地掌握它。

另外一个比较好用的反汇编工具为 [jclasslib](https://github.com/ingokegel/jclasslib/releases)。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

在IDEA 插件中心中还可以搜到该工具的IDEA插件。

当然，还有很多其他好用的 Java 反编译和反汇编软件，希望大家平时多尝试，多练习。

希望大家能够熟练掌握其中一两种，能够快速反编译和反汇编，帮助自己学习知识和解决问题。



#### 4.效率软件



##### 4.1 效率



###### 4.1.1 Alfred

Alfred 可以说是 Mac 系统的效率神器。该软件支持文件搜索、粘贴板管理、快捷短语提示、各种工作流等功能。

具体功能介绍可以看[这篇文章](https://sspai.com/post/55098)。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



###### 4.1.2 Wox

有些朋友可能会说，我们系统是 windows 的肿么办？

这里推荐一个 windows 上的 alfred： [Wox](http://www.wox.one/)， 该软件支持软件、文件、浏览器书签等搜索，支持通过快捷键快速搜索网页，还支持丰富的插件，可以查询英语单词、查快递等。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

软件效果图（图片来自官网）



###### 4.1.3 Snipaste

另外推荐一个非常好用的截图和贴图软件 [Snipaste](https://zh.snipaste.com/)。

该软件不仅是一款截图工具，还支持将截图贴到屏幕上，使用非常简单， F1 截图，然后 F3 贴图，截图就会桌面置顶显示。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

软件效果图（图片来自官网）

我们写技术文章或者开发时可能需要参考多个地方，由于开发的桌面和参考的桌面通常不在一个桌面，如果没有双屏，或者双显示屏还不过还需要切换，就非常浪费时间。此时该软件就有大用途，可以将待参考的内容分别截图、贴图，然后自己随意排列组合在当前页面中供你参考。



###### 4.1.4 Contexts

该软件目前只支持 mac 系统，可以实现窗口的快速切换。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

软件效果图（图片来自官网）



###### 4.1.5 Paste

该软件目前只支持 mac 系统。

采用 iOS 多任务卡片切换界面，可以可视化粘贴板历史，支持剪切搜索，热键快速调用，可以快速选取想要的粘贴版历史内容并粘贴到当前应用中。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 4.2 画图神器

作为一个合格的程序员，怎能没几个趁手的画图工具呢？

每个人的喜好各有不同，下面推荐几个本人和身边人开发中常用的画图工具。



###### 4.2.1 UML 画图工具

**visual-paradigm**

推荐 [visual-paradigm](https://online.visual-paradigm.com/cn/diagrams/tutorials/)的理由是该画图工具不仅支持软件本地画图，还支持在线画图，支持最新的语法，并且有丰富的参考示例。

**PlantUML**

强烈推荐大家画UML 图时使用[PlantUML](http://plantuml.com/zh/index)，理由是其他大多数作图软件都采用拖拽式，对于有些强迫症的人会浪费很多时间进行对齐等操作。

该软件还提供了 IDEA 插件，在IDEA中创建 plantUML 的图形支持实时预览。

通过 PlantUML 官网给出的示例，大家可以快速上手。

**其它UML画图工具**

可以使用 processon 来作图，优势是在线存储。windows 系统用户可以使用 visio，功能强大，画的图也很美观。



####4.2.2 思维导图

很多人会有些奇怪，为啥推荐思维导图呢？

其实对于Java工程师来说，思维导图是梳理知识，梳理需求的重要工具。

然而画思维导图并不是照着目录列一遍，而是带上自己的思考，具体再画图篇会讲到。

思维导图软件推荐使用： xmind、mindjet、ithoughts 等。



##### 4.3 辅助开发



###### 4.3.1 PostMan

[PostMan](https://www.getpostman.com/) 可以模拟前端请求，可以将请求进行分类、保存，支持变量，支持将请求导出为 curl 等其他请求方式，功能非常强大，大家可以根据官方文档多摸索使用。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



###### 4.3.2 VisulVM

[VisulVM](https://visualvm.github.io/) 是 JDK 命令行工具的可视化整合工具，可以在开发和生产中使用。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

常规用法是先启动软件，然后选择本地的 Java 进程，或者添加远程机器的 Ip 和端口号监控远程 Java 进程状况。

IDEA 中还有 [VisualVM 的插件](https://plugins.jetbrains.com/plugin/7115-visualvm-launcher/)，可以在本地应用启动时，自动启动 VisualVM。



###### 4.3.3 前端插件助手

推荐一个方便大家开发的插件：[前端插件助手](https://www.baidufe.com/)。

该插件支持字符串的编解码、JSON串的格式化、代码美化、二维码生成器、页面滚动截屏、图片转Base64 、简易 Postman、Ajax 调试等功能。

虽然名叫“前端插件助手”，其实该插件对我们后端开发帮助也极大。



###### 4.3.4 Print Friendly & PDF

我们平时看很多博客等，想保存为PDF，如果直接使用浏览器打印就会发现有很多广告等信息。

可以使用该插件，生成只包含页面主要内容的 PDF。

大家可以通过该软件的[官网](https://www.printfriendly.com/) 进一步了解该插件。



###### 4.3.5 ModHeader

该插件可以修改请求和响应头，在某种调试场合非常有用。



###### 4.3.6 Ajax Interceptor

[该插件](https://github.com/YGYOOO/ajax-interceptor)非常强大，可以修改页面 Ajax 请求的返回结果。



###### 4.3.7 翻译插件

**沙拉查词**

很多同学想看英文技术网站，但是英语不是特别好，可以借助该插件聚合多种翻译软件，翻译各种词汇或句子。

最大的好处是可以对比多种翻译插件的结果，得到最准确的理解。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

**彩云翻译**

[彩云翻译](https://fanyi.caiyunapp.com/#/web) 提供了中英文对照翻译的能力，如果看某些英文技术文章有些吃力，可以适当使用该插件实现中英文对照理解。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



#### 5.很赞的网站



##### 5.1 在线练习网站

很多人想学习某个技术，但是有自己电脑配置限制或者嫌麻烦等各种原因，可能不愿意安装某些环境。

**那么有没有可以在线练习的网站呢？** 答案是：有。

接下来推荐几个非常强大的在线练习和学习网站。



###### 5.1.1 Git 在线练习

推荐一个在线学习 Git 的趣味网站: https://learngitbranching.js.org/
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



###### 5.1.2 kafka集群体验

有一个网站提供kafka集群的体验：https://www.cloudkarafka.com/



###### 5.1.3 leetcode

此处，不得不提的是鼎鼎大名的 [leetcode](https://leetcode.com/)。

该网站提供了在算法、数据库和Shell 脚本的练习题。



###### 5.1.4 数据结构可视化

接下来推荐一个[数据结构可视化](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)的网站。可以选择某种数据结构，动态添加数据，观察变化过程。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



###### 5.1.5 正则表达式

还有很多支持在线学习和验证正则表达式的网站，如 https://regexr.com/、 https://c.runoob.com/front-end/854、https://tool.oschina.net/regex。



###### 5.1.6 在线练习SQL

推荐几个可在线练习 SQL 的超赞网站：[SQLZOO](https://sqlzoo.net/)、[SQLBolt](https://sqlbolt.com/)、[SQL Fiddle](http://sqlfiddle.com/)。

中文版：[xuesql](http://xuesql.cn/)、[廖雪峰SQL教程](https://www.liaoxuefeng.com/wiki/1177760294764384/1179610846971200)

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 5.2 实用网站



###### 5.2.1 时间戳转换

时间戳转换工具：https://tool.chinaz.com/tools/unixtime.aspx



###### 5.2.2 JSON相关

**JSON格式化**

开发中还会经常用到格式化 JSON 串的功能，[bejson](https://www.bejson.com/) 提供了 JSON相关的丰富功能，JSON的格式化校验、压缩、转义、去除转义等。

** JSON 和 Java实体互转**

有很多强大的网站支持 JSON和Java实体互转，如 [bejson](https://www.bejson.com/json2javapojo/new/)、[jsonschema2pojo ](http://www.jsonschema2pojo.org/)、[codebeautify](https://codebeautify.org/json-to-java-converter)、[FreeCodeFormat](https://www.freecodeformat.com/json2pojo.php)、[site24*7](https://www.site24x7.com/tools/json-to-java.html) 等。



###### 5.2.3 超赞的英文 Java学习网站

除了咱们的慕课网外，推荐几个非常好的英文学习网站。

首推 [baeldung](https://www.baeldung.com/category/java/)该网站几乎所有的文章都有[配套代码](https://github.com/eugenp/tutorials),。我们可以直接通过该网站的代码运行学习某些知识点，某些框架等。

其次是 [javacodegeeks](https://www.javacodegeeks.com/), 该网站会提供丰富的 Java 教程，还会提供一些英文 PDF 教程。

[journaldev](https://www.journaldev.com/) 和 [jamesdbloom](http://blog.jamesdbloom.com/) 对技术的讲解非常透彻。



###### 5.2.4 技术电子书百宝箱

Library Genesis 号称是帮助全人类知识传播计划，其网站 http://gen.lib.rus.ec/ 提供了很多英文图书的下载。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

我们 Java 开发需要用到知名英文书籍几乎都可以在上面找到电子版。

强烈建议大家购买纸质版经典的 Java 技术图书，反复学习。



###### 5.2.5 GitHub

GitHub 也可堪称是百宝箱，大家可以通过它来搜索想学技术的源码和相关示例代码。

大家可以在 Java 的 [topic栏目](https://github.com/topics/java) 了解 stars 最多的，最近更新的，最佳的 Java项目等。



#### 6.总结

本文重点介绍了 Java 学习和工作中常用的软件、插件、网站等。熟练地使用这些工具，将有助于提高我的开发效率和编程体验。

肯定还有很多好用的插件和软件，由于篇幅有限就不在这里一一介绍，欢迎大家留言分享。

希望通过本小节的介绍能够助力大家的学习和进阶。

如果你觉得本专栏对你有帮助，欢迎推荐给更多朋友，一起交流学习。



## 异常日志



### 一些异常处理建议



#### 1.前言

《手册》的异常处理规约对异常的处理方式提出了一些指导规范[1](https://www.imooc.com/read/55/article/1155#fn1)，如：

> 【强制】异常不要用来做流程控制，条件控制。
>
> 【强制】有 try 块放到了事务代码中，catch 异常后，如果需要回滚事务，一定要注意手动回滚事务。

常规try - catch 捕捉异常非常简单，相信大家都很熟悉，这里就不作展开。

本节重点讲述一些异常处理姿势不正确导致的各种 BUG， 并给出一些异常处理的建议。



#### 2.要不要"吞掉"异常？

在实际开发中关于异常的一个重要问题是：要不要“吞掉”异常。

所谓 “吞掉” 异常是指：处理后不再将异常传给上层。其中包括 catch 到异常并处理（打印日志、发通知等）后不再扔给上层；捕捉到异常后给上层返回 null 值等行为。

其中 “**有 try 块放到了事务代码中，catch 异常后，如果需要回滚事务，一定要注意手动回 滚事务**。” 就属于其中一例。

**那么为什么要手动回滚呢？**

我们看下事务的执行入口：

```
TransactionInterceptor#invoke
```

调用到了这里 `TransactionAspectSupport#invokeWithinTransaction` ：

```java
/**
	 * General delegate for around-advice-based subclasses, delegating to several other template
	 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
	 * as well as regular {@link PlatformTransactionManager} implementations.
	 * @param method the Method being invoked
	 * @param targetClass the target class that we're invoking the method on
	 * @param invocation the callback to use for proceeding with the target invocation
	 * @return the return value of the method, if any
	 * @throws Throwable propagated from the target invocation
	 */
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		// 省略

	}
```

带 @Transaction 注解的事务函数中捕获到异常后，执行如下代码：

```
TransactionAspectSupport#completeTransactionAfterThrowing
/**
 * Handle a throwable, completing the transaction.
 * We may commit or roll back, depending on the configuration.
 * @param txInfo information about the current transaction
 * @param ex throwable encountered
 */
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
      }
      else {
         // We don't roll back on this exception.
         // Will still roll back if TransactionStatus.isRollbackOnly() is true.
         try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
      }
   }
}
```

可以看到，如果设置了事务属性且当前异常满足 rollbackOn 指定的异常（默认为 RuntimeException 类型及其的子类以及Error 及其子类），则会将当前事务回滚，否则提交。

因此如果 catch 异常后没有再次将异常抛出或者不手动回滚，将会导致事务提交。

**在封装二方接口很多人也会选择 “吞掉” 异常**，示意代码如下：

```java
@Component
public class DemoClient {

    @Resource
    private XXServcie xxServcie;

    public XXInfo someMethod(Long id) {
        try {
            return xxServcie.getXXInfo(id);
        } catch (Exception e) {
            log.warn("调用xx服务异常，参数:{}", id, e);
            return null;
        }
    }
}
```

当调用发生异常时打印异常信息后直接返回 null。

此时如果上层调用方直接拿到返回值对象未做判空处理直接使用其属性，很容易报 NPE。

另外由于此处 “吞掉” 了二方接口的异常，有些业务异常中包含的错误原因（如包含xxx敏感词汇、标题不能超过20个字等）无法传给上层再封装给前端，可能会造成出错后用户懵逼，增加很多用户咨询。

比如用户输入了某个敏感词汇，调用二方接口时 “吞掉” 了敏感词汇的业务异常提示（输入中包含 xx敏感词），用户通过技术支持咨询，开发人员要查日志才能知道具体的错误原因（如果此处没打印日志，可能连日志都没得查），非常低效。

开发中要根据具体业务场景慎重确定是否要“吞掉” 异常，一个 “不经意” 的写法可能会造成很多线上咨询甚至线上 BUG。



#### 3.循环中的异常处理问题

**【参考】特别注意循环的代码异常处理的对程序的影响。**

我们先看下面的例子：

```java
   public static void main(String[] args) throws InterruptedException {

        List<String> data = new ArrayList<>();
        data.add("a");
        data.add("ab");
        data.add("abc");
        data.add("abcd");

        for (String str : data) {
            // 远程方法调用
            String result = doSomeRemoteInvoke(str);
            System.out.println(result);
        }
     
     // 后续代码
    }
```

在写代码时这种场景非常常见，需要注意的是如果不对循环代码进行捕捉，如果循环中出现异常，后续代码则无法执行。

但是如果在 for 循环外部捕捉异常，虽然for循环后如果有代码依然可以执行，但是列表中的非最后一个元素作为参数调用 `doSomeRemoteInvoke` 出现异常，后续数据无法继续执行。

```java
try {
    for (String str : data) {
        // 远程方法调用（可能出现异常）
        String result = doSomeRemoteInvoke(str);
        System.out.println(result);
    }
} catch (Exception e) {
    log.error("程序出错，参数data:{},错误详情", JSON.toJSONString(data), e);
}
```

因此需要对 for 循环代码内对可能出现的异常进行捕捉：

```java
for (String str : data) {
    try {
        // 远程方法调用（可能出现异常）
        String result = doSomeRemoteInvoke(str);
        System.out.println(result);
    } catch (Exception e) {
        log.error("程序出错，参数data:{},错误详情", JSON.toJSONString(data), e);
    }
}
```

我们再看下面一个例子，思考两个问题：

分别调用两个函数 `pirntList1` 和 `printList2` 输出的结果有何不同？

哪个不需要捕捉异常也不会造成中间有一个出错后续处理中断？

代码如下：

```java
public class ExceptionDemo {

    public static void main(String[] args) throws InterruptedException {

        ExecutorService executorService = Executors.newSingleThreadExecutor();
        List<String> data = new ArrayList<>();
        data.add("a");
        data.add("ab");
        data.add("abc");
        data.add("abcd");

        printList1(data, executorService);
       // printList2(data, executorService);
    }

    private static void printList1(List<String> data, ExecutorService executorService) {
        for (String str : data) {
            executorService.execute(() -> {
                // 模拟中间报错
                if (str.length() == 2) {
                    throw new IllegalArgumentException();
                }
                System.out.println(str);
            });
        }
    }

    private static void printList2(List<String> data, ExecutorService executorService) {
        executorService.execute(() -> {
            for (String str : data) {
                // 模拟中间报错
                if (str.length() == 2) {
                    throw new IllegalArgumentException();
                }
                System.out.println(str);
            }
        });
    }
}
```

**让我们来分析这两个函数的区别**：

在函数 `pirntList1` 中和上面的代码非常相似，for 循环在线程池执行代码外部，每次循环调用线程池去执行判断和打印语句。

此时依次传入 a、ab、abc、abcd 四个字符串；当执行到 ab 时会抛出 `IllegalArgumentException`，此时线程池中的唯一的线程销毁；当执行到 abc 字符串时，再次在线程池中执行，线程池创建新的线程来执行，依然可以正常执行。

![图片描述](img/5ddb40b900016ed408570217.png)

而在函数 `pirntList2` 中 for 循环在 线程池 `execute` 参数的lambda表达式内，所有的循环执行都在同一个线程内。当执行到 ab 字符串时，抛出了异常，导致整个线程销毁，无法继续执行。
![图片描述](img/5ddb40a80001334608860224.png)

因此为了不让一个数据出错导致后续的代码都无法执行，如果采用第二种方式来执行可以对代码做出如下修改：

```java
private static void printList2(List<String> data, ExecutorService executorService) {
    executorService.execute(() -> {
        for (String str : data) {
            try {
                // 模拟中间报错
                if (str.length() == 2) {
                    throw new IllegalArgumentException();
                }

                System.out.println(Thread.currentThread().getName() + "-->" + str);
            } catch (Exception e) {
                log.error("程序出错，参数data:{},错误详情", JSON.toJSONString(data), e);
            }
        }
    });
}
```

在实际业务开发过程中，这种问题比较隐蔽，尤其是在异步线程中执行时，如果不加留意，很容易出现上面所描述的问题。



#### 4.补充

**【建议】慎重思考是否“吞掉” 异常，在二方服务封装时，如捕捉异常，应打印出查询参数和异常详情。**

实际开发中，一般都不会吞掉异常，遇到 “吞掉” 异常的场景要慎重思考是否合理。

另外，正如第二部分给出的范例所示，如果调用二方接口出现异常没有打印日志，将对排查问题造成很大的困难。

**【建议】要理解好受检异常和非受检异常的区别，避免误用。**

Java 中的异常主要分为两类：受检异常和非受检异常。

根据 JLS 异常部分的[相关描述](https://docs.oracle.com/javase/specs/jls/se9/html/jls-11.html#jls-11.1)，我们可知受检异常主要指编译时强制检查的异常，包括非受检异常之外的其他 `Throwable` 的子类；非受检异常主要指编译器免检异常，通常包括运行时异常类和 Error相关类。

![图片描述](img/5ddb408d0001a14017041002.png)
Error 和 Exception 都是 Throwable的子类。 RuntimeException 和其子类都属于运行时异常。Error 类和其子类都属于错误类。RuntimeException 及其子类 和 Error类及其子类 属于非受检异常，除此之外的 其他 Throwable 子类属于受检异常。

大家可以使用 IDEA 自带的类图功能，绘制出自己感兴趣的异常类型，通过上述原则分析其是否属于受检异常。

通常开发中自定义的业务异常（BusinessException）属于非受检异常，会定义为 RuntimeException 的子类。

有些朋友可能会将业务异常定义为受检异常，导致底层抛出后上层调用每层都要被迫处理它。

**【建议】努力使失败保持原子性。**

正如《Effective Java》第 3 版 第 76条 努力使失败保持原子性[^2] 所提到的那样。

我们可以在函数核心代码执行前对参数进行检查，对不满足的条件抛出适当的异常。

实际开发中通常可以使用 `com.google.common.base.Preconditions` 或者 `org.apache.commons.lang3.Validate` 第三方库提供的参数检查工具类来实现。

**【建议】如果忽略异常，请给出理由**

如果 catch 住异常却没有进行编写任何处理代码，请在注释中给出充分的理由，避免其他人产生困惑，避免留坑。

大家可以参考 `org.springframework.context.support.AbstractApplicationContext#close` 源码：

```java
@Override
public void close() {
   synchronized (this.startupShutdownMonitor) {
      doClose();
      // If we registered a JVM shutdown hook, we don't need it anymore now:
      // We've already explicitly closed the context.
      if (this.shutdownHook != null) {
         try {
            Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
         }
         catch (IllegalStateException ex) {
            // ignore - VM is already shutting down
         }
      }
   }
}
```

上面的源码捕捉到 `IllegalStateException` 异常以后没有处理，给出了处理方式和原因: 忽略此异常，因为虚拟机已经正在关闭。



#### 5.总结

本节主要讲异常的一些处理建议，包括是否要 “吞掉” 异常，循环中的异常处理，以及一些补充建议。希望大家可以重视异常，少趟坑。

下一节我们将讲解打印日志的目的，该打印哪些日志，不该打印哪些日志以及忘记打印日志该怎么办等问题。



### 日志学习和使用的正确姿势



#### 1. 前言

日志虽然表面看起来很简单，却非常重要，日志是我们排查问题的非常重要的手段。我们不仅要掌握日志的基本用法，更要懂得不该在哪里打日志，该在哪些地方打日志，并思考忘记打日志怎么办。

《手册》第 26 到 27 页，提出了很多日志规约 [1](https://www.imooc.com/read/55/article/1156#fn1)，其中比较重要的有：

> 【强制】应用中不要直接使用日志系统的 API，而是应该依赖日志架构 SLF4J 中的 API，使用门面模式的日志架构，有利于维护各个类的日志处理方式统一。
>
> 【强制】日志至少要保留 15 天，因为有些异常具备以 "周" 为频次的特点。
>
> 【强制】避免重复打印日志，浪费磁盘空间，务必在 log4j.xml 中设置 additivity =false。
>
> 【强制】异常信息应该包括两类信息：案发现场信息和异常堆栈信息。
>
> 等。

看到这些我们该思考下面几个问题：

- 我们门面模式的是什么？它的使用场景和优势是什么？
- 为什么会重复打日志？
- 不该在哪里打日志？
- 该在哪些地方打日志呢？
- 如果忘记打日志却着急排查问题怎么办？

只有主动去思考和日志规约相关的问题，这样才能知其然，才能学得更多更深入，在使用时才能更灵活。

本节将带领大家学习和分析这些问题。



#### 2. 学习日志



##### 2.1 日志是什么？为什么要打印日志？

**日志文件是什么？**

计算机领域，日志文件是记录发生在运行中的操作系统或其他软件中的事件或消息的文件。

**为什么要打印日志？**

打印日志的主要目的是为了监测系统状态、方便测试、方便排查问题。

当测试时遇到和预计不符的情况，看日志是解决问题的最常用手段。设想一下如果没有日志，线上出现问题排查起来是不是更困难？

很多监控系统都是通过监控日志来预警，很多线上问题通过日志来排查，很多测试人员依靠日志来辅助测试。



##### 2.2 门面模式

很多同学可能有这样的一种体会：专门去学设计模式，看会了也容易忘。其实带着问题学知识，印象会更加深刻。

比如此日志规约章节就涉及到了门面模式，那么这就是我们学习和理解门面模式的好机会。

学习设计模式我们主要思考以下几个问题：

- 为什么会出现这种设计模式？
- 这种设计模式的核心思想是什么？
- 这种设计模式的使用场景有哪些？
- 这种设计模式的优缺点有哪些？

那么我们依次来回答这几个问题，我们可以参考《设计模式之禅》，可以参考[菜鸟教程](https://www.runoob.com/design-pattern/facade-pattern.html)，甚至直接通过搜索博客。



###### 2.2.1 门面设计模式是为了解决什么问题呢？

《代码整洁之道》第二章 有意义的命名，讲到：命名要名副其实。

> 我们想要强调，这件事很严肃。选一个好名字要花时间，但是生下来的时间比花费的时间多。注意命名，而且一旦发现有更好的名称，就换掉旧的。这么做，读你代码的人（包括你自己）都会更开心。

同样地，对于门面设计模式，顾名思义，是为了隐藏系统的复杂性，向使用方提供统一的可以访问系统的接口。



###### 2.2.2 门面模式的核心思想是什么？

门面设计模式在客户端和复杂的系统之间加一层，在这一层将调用的顺序和依赖的关系调整好。



###### 2.2.3 门面模式的主要使用场景有哪些？

为复杂的子系统提供外界访问的模块。

预防低水平的开发人员带来的风险。



###### 2.2.4 门面模式的优缺点分别是什么？

**优点**：减少了系统的相互依赖；提高了灵活性和安全性。

**缺点**：不符合开闭原则，如果接口要修改无法通过继承和覆写来解决，只能修改门面代码，影响面比较大。



###### 2.2.5 再回到为什么应用应该依赖使用日志架构 SLF4J 中的 API，使用门面设计模式的日志架构？

SLF4J 的全称为： The Simple Logging Facade for Java， 即 Java 简单日志门面。

使用 SLF4J 编写代码，开发人员不需要关注不同的日志框架的差异，各日志框架对 SLF4J 做适配。由于没有具体依赖某个日志框架，如果系统出于安全、性能等原因想更换另外一个新的日志框架就轻而易举。

关于门面设计模式的具体编码，大家可以参考《设计模式之禅》的 23 章：门面模式。



##### 2.3 日志级别



###### 2.3.1 日志级别规范

常用的日志级别分为：ERROR、WARN、INFO、DEBUG、TRACE。

**ERROR 日志**的使用场景是：影响到程序正常运行或影响到当前请求正常运行的异常情况。比如打开配置失败、调用二方或者三方库抛出异常等。

**WARN 日志** 的使用场景是：不应该出现，但是不影响程序正常运行，不影响请求正常执行的情况。如找不到某个配置但是使用了默认配置，比如某些业务异常。

**INFO 日志**的使用场景是：需要了解的普通信息，比如接口的参数和返回值，异步任务的执行时间和任务内容等。

**DEBUG 日志**的使用场景是：所有调试阶段想了解的信息。比如无法进行远程 DEBUG 时，添加 DEBUG 日志在待研究的函数的某些位置打印参数和中间数据等。

如 Spring `org.springframework.boot.SpringApplication#load` 函数就用到了 DEBUG 日志：

```java
/**
 * Load beans into the application context.
 * @param context the context to load beans into
 * @param sources the sources to load
 */
protected void load(ApplicationContext context, Object[] sources) {
   if (logger.isDebugEnabled()) {
      logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
   }
   BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
   if (this.beanNameGenerator != null) {
      loader.setBeanNameGenerator(this.beanNameGenerator);
   }
   if (this.resourceLoader != null) {
      loader.setResourceLoader(this.resourceLoader);
   }
   if (this.environment != null) {
      loader.setEnvironment(this.environment);
   }
   loader.load();
}
```

**TRACE 日志** 的使用场景是：非常详细的系统运行信息，比如某个中间件读取配置，启动完成等。

如 Spring 源码中，`org.springframework.boot.env.RandomValuePropertySource#getProperty` 就打印了 TRACE 级别的日志：

```java
@Override
public Object getProperty(String name) {
   if (!name.startsWith(PREFIX)) {
      return null;
   }
   if (logger.isTraceEnabled()) {
      logger.trace("Generating random property for '" + name + "'");
   }
   return getRandomValue(name.substring(PREFIX.length()));
}
```

实际业务开发中 TRACE 级别的日志很少使用。

另外通过上面两个例子，我们看到调用打印语句前，都会先判断该级别的日志是否开启。大家可以先思考下为什么这么做？文章后半部将重点对此进行解释。



###### 2.3.2 规范的日志级别

规范打日志可以让根据日志定位问题的同学能够抓住重点，比如优先关注错误日志，其次是警告。

【推荐】在自测或提测之后上线前一定要注意 warn 级别以上的日志，特别是 error 日志。

通常我们会将 ERROR 日志专门输出到一个 error.log 文件。调试时通过 `tail -f error.log` 随时监控出现的错误日志。

希望大家**一定**要养成这种好习惯。通过这种习惯，可以尽可能早地发现问题，避免悲剧。

> 有些报错虽然影响实际的功能，但是由于不影响主流程，很多人就没在意。
>
> 实际开发中，由于习惯查看错误日志，某次自测时发现某个刚上线的功能的某项配置错误，导致某项功能后台启动失败，及时反馈给该服务的负责人，收到了点赞。



###### 2.3.3 不规范日志级别带来的问题

日志级别看似非常简单，但是很多人可能会打错日志级别，给调试和定位问题带来很大不便。

比如在实际开发中，发现一个**功能失败，开发人员居然打印的是 info 级别日志**！

某项功能失败，却找不到任何错误和警告级别的日志，坑队友…

还有将严重的错误打印 WARN 级别日志，导致没有及早引起重视出现故障。



#### 3. 举例

日志的最基本用法比较简单，大家自行学习，这里就不再举例了。这里举两个例子来说明如何学习日志相关知识，如何分析相关问题。



##### 3.1 叠加性如何理解？

前面提到：为了避免重复打印日志，浪费磁盘空间，务必在 `log4j.xml` 中设置 `additivity =false`。

**那么为什么会重复打印日志呢？**

我们分别从官方文档、源码的角度对该问题进行学习。



###### 3.1.1 官方文档大法

这类问题我们需要从官方文档或者源码层面去寻找答案。

在 [log4j 的官方手册的 “配置” 章节](http://logging.apache.org/log4j/2.x/manual/configuration.html) 中的一个案例：

`MyApp` 类：

```java
import com.foo.Bar;
 
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
 
public class MyApp {
 
    private static final Logger logger = LogManager.getLogger(MyApp.class);
 
    public static void main(final String... args) {
 
        logger.trace("Entering application.");
        Bar bar = new Bar();
        if (!bar.doIt()) {
            logger.error("Didn't do it.");
        }
        logger.trace("Exiting application.");
    }
}
```

`Bar` 类：

```java
package com.foo;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;
 
public class Bar {
  static final Logger logger = LogManager.getLogger(Bar.class.getName());
 
  public boolean doIt() {
    logger.entry();
    logger.error("Did it again!");
    return logger.exit(false);
  }
}
```

如果配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

会输出这样的结果：

```java
15:12:13.226 [main] TRACE com.foo.Bar - Enter
15:12:13.226 [main] TRACE com.foo.Bar - Enter
15:12:13.229 [main] ERROR com.foo.Bar - Did it again!
15:12:13.229 [main] ERROR com.foo.Bar - Did it again!
15:12:13.229 [main] TRACE com.foo.Bar - Exit with(false)
15:12:13.229 [main] TRACE com.foo.Bar - Exit with(false)
15:12:13.230 [main] ERROR MyApp - Didn't do it.
```

通过该范例和文档的描述，我们可以看到 `com.foo.Bar` 的消息都输出了两次。

这是因为第一次是使用到了 `com.foo.Bar` 这个 logger，然后又用到了它的父 logger。

日志事件被传递到 root logger 的 appender ，被写到了 console 中，导致两次输出， 这就是所谓的叠加性。

叠加性是一个非常方便的特性，低层次的 logger 甚至都不需要配置 appender，就可以输出。但是很多情况下并不希望有这种默认的行为，那么可以通过设置 logger 的 additivity 属性为 false 来关闭叠加性。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace" additivity="false">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

**那么 logback 是否也提供了类似的设置呢？**

我们查看官方手册的 [Appenders and Layouts”](https://logback.qos.ch/manual/architecture.html) 章节提到：

> 一个 logger 可以被关联到多个 appender 上，对于 logger 的每个启用了的记录请求，都将被发送到 logger 里的全部 appender 及更高层次的 appender 中。换句话说，appender 叠加地继承了 logger 的层次等级。如果 root 日志有一个控制台 console ，那么所有启动的日志至少都会输出到控制台中。如果 root 下还有一个叫 L 的 logger，它含有一个文件 appender，那么 L 和 L 的子层次 Logger 的所有日志都将会打印到控制台和文件中。
>
> 将 logger 的 additivity 属性设置为 false ，则可以取消这种默认的 appender 累加行为。

另外关于 Appender 的叠加性还有如下描述：

Appender 叠加性的含义是：Logger L 的日志的输出会发送给 L 及其祖先的全部 appender。

然而，如果 Logger L 的某个祖先 Logger P 设置叠加标识为 false，那么，Logger L 的输出会发送给 Logger L 和 Logger P (含 P) 的所有 appender，但是不会发送给 Logger P 的任何祖先的 appender。

这一点和 log4j 非常相似。



###### 3.1.2 源码模式

我们如果使用 log4j（logback 会略有不同，但是思路都是相似的），通过断点调试我们发现代码的核心源码如下：

```
org.apache.logging.log4j.core.config.LoggerConfig#log
protected void log(final LogEvent event, final LoggerConfigPredicate predicate) {
    if (!isFiltered(event)) {
        processLogEvent(event, predicate);
    }
}
```

`processLogEvent` 函数源码:

```java
private void processLogEvent(final LogEvent event, final LoggerConfigPredicate predicate) {
   event.setIncludeLocation(isIncludeLocation());
   if (predicate.allow(this)) { 
       callAppenders(event); // 调用 appender 写出日志
   }
  logParent(event, predicate); // 让父 logger 记录日志
}
```

其中的 `callAppenders` 函数源码：

```java
protected void callAppenders(final LogEvent event) {
    final AppenderControl[] controls = appenders.get();
    //noinspection ForLoopReplaceableByForEach
    for (int i = 0; i < controls.length; i++) {
        controls[i].callAppender(event);
    }
}
```

如果 additive 为 true 且 parent Logger 不为 null，则调用 parent 的 log 函数。

```java
private void logParent(final LogEvent event, final LoggerConfigPredicate predicate) {
    if (additive && parent != null) { 
        parent.log(event, predicate); // 调用父日志继续打印
    }
}
```

从这几个核心函数我们可以看出，如果 additive 为 false，则不会传递，parent Logger 不会继续记录日志。

很多人学到此可能会有些不以为意，因为很多人加入团队后一般都是维护现有的项目，没有机会去配置日志，也没注意去观察日志文件的区别。

然而在工作中的确发现有的服务 root 日志超大，很多日志被重复打印，造成了资源的浪费的情况。



##### 3.2 为什么推荐使用占位符方式打印日志？

《手册》中规定有一条关于占位符的规定：

> 【强制】在日志输出时，字符串变量之间的拼接使用占位符的方式
>
> 说明：因为 String 字符串拼接会使用 StringBuilder 的 append () 方式，有一定的性能损耗。使用占位符可以有效提高性能。

很多人会把这一段话当做说服自己或者别人使用占位符的依据，然后就没然后了…

如果我们遇到类似的新问题，或者我们没看到《手册》中这条规定，我们如何学习和分析？

- **我们怎么知道 “String 字符串拼接会使用 StringBuilder 的 append () 方式”？**
- 俗话说：“尽信书不如无书”。规约中这种说法严谨吗？真的都是使用这一种方式拼接字符串的吗？

我们写一个简单的 DEMO:

```java
@Slf4j
public class LogDemo {

    public void first() {
        log.debug("慕课" + "专栏");
    }

    public void second(String website) {
        log.debug("慕课网" + website);

    }

    public void third(String website) {
        if (log.isDebugEnabled()) {
            log.debug("慕课网" + website);
        }
    }
}
```

对源码编译然后反汇编，得到下面反汇编代码：

```java
public class com.imooc.basic.log.LogDemo {
  public com.imooc.basic.log.LogDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void first();
    Code:
       0: getstatic     #2                  // Field log:Lorg/slf4j/Logger;
       3: ldc           #3                  // String 慕课专栏
       5: invokeinterface #4,  2            // InterfaceMethod org/slf4j/Logger.debug:(Ljava/lang/String;)V
      10: return

  public void second(java.lang.String);
    Code:
       0: getstatic     #2                  // Field log:Lorg/slf4j/Logger;
       3: new           #5                  // class java/lang/StringBuilder
       6: dup
       7: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      10: ldc           #7                  // String 慕课网
      12: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      15: aload_1
      16: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      22: invokeinterface #4,  2            // InterfaceMethod org/slf4j/Logger.debug:(Ljava/lang/String;)V
      27: return

  public void third(java.lang.String);
    Code:
       0: getstatic     #2                  // Field log:Lorg/slf4j/Logger;
       3: invokeinterface #10,  1           // InterfaceMethod org/slf4j/Logger.isDebugEnabled:()Z
       8: ifeq          38
      11: getstatic     #2                  // Field log:Lorg/slf4j/Logger;
      14: new           #5                  // class java/lang/StringBuilder
      17: dup
      18: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
      21: ldc           #7                  // String 慕课网
      23: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      26: aload_1
      27: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      30: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      33: invokeinterface #4,  2            // InterfaceMethod org/slf4j/Logger.debug:(Ljava/lang/String;)V
      38: return

  static {};
    Code:
       0: ldc           #11                 // class com/imooc/basic/log/LogDemo
       2: invokestatic  #12                 // Method org/slf4j/LoggerFactory.getLogger:(Ljava/lang/Class;)Lorg/slf4j/Logger;
       5: putstatic     #2                  // Field log:Lorg/slf4j/Logger;
       8: return
}
```

通过上述反汇编后的代码，我们发现：

如果源码中直接将两个字符串字面量进行拼接，编译时期就会生成新的字符串，并不是通过 StringBuilder 构造的。

上述反汇编后的代码，可以逆向 “翻译” 为：

```java
@Slf4j
public class LogDemo {

    public void first() {
        log.debug("慕课专栏");
    }

    public void second(String website) {
        StringBuilder builder = new StringBuilder();
        builder.append("慕课网");
        builder.append(website);
        String result = builder.toString();
        log.debug(result);
    }

    public void third(String website) {
        if (!log.isDebugEnabled()) {
            return;
        }
        StringBuilder builder = new StringBuilder();
        builder.append("慕课网");
        builder.append(website);
        String result = builder.toString();
        log.debug(result);
    }
}
```

通过反汇编后的代码和上述逆向 “还原” 的 Java 代码，我们可以清楚地知道：如果不加上 `log.isDebugEnabled()` 判断，会因为字符串拼接造成不必要的资源损耗。

但是如果每个日志打印都加上这种判断，代码非常不优雅，因此常见的实现了 `org.slf4j.Logger` 接口的日志框架提供了占位符日志打印方法。

使用占位符的方式，底层会先使用判断逻辑，再去拼接字符串（具体大家可以通过断点调试或者阅读源码来验证），从而避免了不必要的字符串拼接。



#### 3. 不要在哪里打日志？

**【强制】不要用 `System.out.println` 代替日志框架**

很多人，尤其是新手喜欢通过打印语句而不是日志系统来打印日志排查错误。在本地调试的场景下，这种情况更加普遍。这是一个非常不专业且很 low 的行为。

以 Tomcat 为例， 使用 `System.out.println` 函数语句将信息输出到 `catalina.out` 文件中，该文件会被清理，否则影响性能。文件的 IO 操作非常耗时，而该函数底层使用了 `java.io.PrintStream#println(java.lang.Object)` 内部使用了同步代码块，非常影响性能。

由于没有输出级别的概念，很多忘记删除的调试打印语句可能都被输出到输出文件中。如果是必须要打印日志的场景，请用日志框架打印。对于暂时性的本地或者测试服务器上排错，建议使用 IDE 的本地或者远程调试功能。

在调试器中可以看到参数的值，看到调用栈，可以通过表达式查看变量的属性等，功能更加强大。

**【强制】不要打印敏感信息，如果需要打印可以考虑对敏感信息脱敏处理**

很多开发人员没有安全意识，将用户的密码和个人资料、商品的名称和价格等信息在未脱敏的情况下通过日志框架打印到日志文件中。这些都是很大的安全隐患，容易出现安全事故。因此不要打印敏感信息或者将敏感信息进行脱敏后再打印。

**【推荐】除非业务需要，尽量不要打印大文本 (含富文本)。如果要打印可以截取前 M 个字符。**

大家都知道 I/O 操作非常耗时，在高并发场景下，如果同步打印大文本日志非常影响性能。

另外，很多大文本对排查问题帮助不大，打印该信息的意义不大，因此尽量避免打印该内容或只截取一部分关键信息。

如果公司内容有封装的相关注解，可以在大文本上加上忽略的注解。如果没有尽量避免调用 JSON 转字符串的函数将整个对象打印，可以重写 toString 方法忽略或截取大文本字段的一部分。



#### 4. 该在哪里打印日志？

大家写代码时，都会接触到打日志的场景。大多数同学不是不会打印日志，而是不知道在哪打日志。打印日志没有章法，很容易遇到问题才发现没有日志，再去补日志去排查问题，非常浪费时间。

**那么大家是否深入思考过应该在哪里打印日志呢？**

下面给出一些个人建议：

【推荐】**用切面或 Filter 在 dubbo 或 Controller 层做切面来打印调用的参数、返回值和响应时间以及捕捉和打印异常日志。**

通过统一的日志，将参数和返回值，以及响应时间做切面，当出现问题时，可以极大地辅助我们排查，也能够帮助我们了解接口的响应情况。

可以使用日志框架如 CAT，做统一的 traceId, 方便定位整条调用链路。

**【推荐】在依赖的二方或三方接口的参数、返回值和异常处打印日志。**

依赖的二方和三方接口对接最容易出错，可以将其请求的参数和接口返回值以及异常信息等通过日志打印输出，方便分析和排查问题。

一般也会使用切面统一打印，或者通过使用支持日志注解的框架来对某些接口的参数和返回值打印日志。

**【推荐】 在接收消息的地方打印日志。**

我们测试或者排查问题时，如果涉及到消息队列，最先要查的是有没有收到消息，因此接收到消息的地方打印日志比较有用。

**【推荐】 在定时任务的开始和结束的地方。**

和上面的条目很类似，在定时任务的开始和结束的地方打印日志也很重要。

如果线上某个任务没有执行或者开始但是没有结束，可以通过日志快速排查问题。

**【推荐】 在异步任务的开始和结束的地方。**

异步任务如果出错，很难排查。在任务开始和结束的地方打印一些关键参数对排查问题有很大的帮助。

在异步任务的异常捕捉地方打印日志也至关重要，将是分析问题的关键。

**【推荐】面向测试打印日志。**

所谓的**面向测试打印日志**是指：方便测试人员测试开发人员代码而提供的测试。

此日志的级别一般为 DEBUG，一般仅在测试环境可用。

有些测试人员并不会拉取你的代码或者编写单元测试，而只对你的编码进行功能性测试。

此时，测试人员构造测试用例对你的接口进行验证时只能观察到接口的返回值，如果你能提供其中的核心计算函数的日志，将极大方便测试人员去验证功能。



#### 5. 忘记打日志又着急排查问题怎么办？

在开发中我们可能会遇到一种非常尴尬的问题，一个问题本地无法复现，需要去生产环境排查问题。

**但是我们在关键的函数上没有打印日志，肿么办？** 在线等，急…

加上日志后重新发布代价比较大，而且无法快速及时地排查分析问题。

此时我们可以借助 Alibaba Java 诊断利器 [Arthas](https://github.com/alibaba/arthas) 。

安装非常简单，请参考 [Arthas Install](https://alibaba.github.io/arthas/install-detail.html), 一两行命令搞定！启动后选择所要监控的 Java 进程即可进。

如下图所示，我们可以通过 `java -jar arthas-bbot.jar` 启动 arthas。

启动后它会检测已经存在的 java 进程（下图显示有 3 个），选择我们需要研究的进程序号（此处为 1）回车即可。
![图片描述](img/5ddb41080001af2115301004.png)

自此，我们就可以使用其 [watch 命令](https://alibaba.github.io/arthas/watch.html)来观察返回值、抛出的异常、入参等。

**查询出参和返回值**

```
watch 类路径 函数名 "{params,returnObj}" -x 2
```

如输入：watch demo.MathGame primeFactors “{params,returnObj}” -x 2

```java
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 68 ms.
ts=2019-11-09 22:14:05; [cost=2.149321ms] result=@ArrayList[
    @Object[][
        @Integer[70214],
    ],
    @ArrayList[
        @Integer[2],
        @Integer[35107],
    ],
]
ts=2019-11-09 22:14:06; [cost=0.200091ms] result=@ArrayList[
    @Object[][
        @Integer[119344],
    ],
    @ArrayList[
        @Integer[2],
        @Integer[2],
        @Integer[2],
        @Integer[2],
        @Integer[7459],
    ],
]
```

可以试试观察到 `MathGame` 类的 `primeFactors` 函数调用的出参和返回值。

**查看异常信息**

格式如下：`watch 类路径 函数名 "{params[0],throwExp}" -e -x 2`

使用 watch demo.MathGame primeFactors “{params [0],throwExp}” -e -x 2 可以抓取到官方案例的该函数调用的异常信息:

```java
ts=2019-11-09 22:18:36; [cost=0.211415ms] result=@ArrayList[
    @Integer[-26077],
    java.lang.IllegalArgumentException: number is: -26077, need >= 2
	at demo.MathGame.primeFactors(MathGame.java:46)
	at demo.MathGame.run(MathGame.java:24)
	at demo.MathGame.main(MathGame.java:16)
,
]
```

watch 命令还支持条件表达式，支持根据耗进行过滤等强大功能，建议大家一定要敢于尝试，通过官方案例或自己项目代码去训练。



#### 6. 常见错误日志形式

下面给出实际开发中的典型误用的几种情况：

**用打印语句或者 `e.printStackTrace()` 打印日志**

这两种打印方式非常不专业，容易造成日志丢失，污染控制日志级别，甚至还可能造成其他问题。

**占位符误用**

```java
log.info("错误信息为={}", e);
```

对应的函数签名为：

```java
public void info(String msg, Throwable t);
```

最终导致占位符被当做普通字符串处理。

**参数格式错误**

```java
log.info( user.getAge()+"", user.getName());
```

此时实际对应的函数为：

```java
public void info(String format, Object arg);
```

由于第一个 format 参数没有格式化占位符，导致第二个参数打印不出来。

**空指针异常**

实际开发中，还有一些同学居然会这么写

```java
if (result == null || result.getData() == null) {
    log.info("result={},resultData:{}", result, result.getData());
}
```

如果 result == null 为 true，难道不会报空指针异常吗？

还有很多花式踩坑姿势，这里就不一一列举了，也欢迎大家留言补充。

建议大家一定要注重 IDE 的警告；开发时日志的函数有较多重载方法，没把握时可以点到源码中核对；多使用 findbugs 来检查代码。



#### 7. 总结

本节我们我们重点分析了如何学习日志规约，思考了为何要打印日志，并讲述了不该在哪里打印日志，该在哪里打印日志的问题。还介绍了如果没有日志还需要紧急排查问题，大家可以使用 arthas 来实现；最后给出了常见的日志形式。

希望大家在学习和开发过程中多思考，多实战，能够够举一反三。

建议大家多读读 log4j 和 logback 的官方手册和源码，更系统地学习日志的用法、理解日志的原理。

下一节我们将学习单元测试相关知识。



#### 8. 课后思考

1、为什么 Spring 中会有下面代码这种先判断后打印日志的情况，而不直接使用占位符方式打印日志呢？

```java
if (logger.isDebugEnabled()) {
   logger.debug(
         "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
}
```

2、学习 `System.out.println` 的源码



## 单元测试



### 单元测试的知识储备



#### 1. 前言

单元测试作为编码质量的重要保障手段，是编码的一个非常重要的环节。

《手册》 第三部分对单元测试进行了描述 [1](https://www.imooc.com/read/55/article/1157#fn1)，包括：单元测试必须遵守自动化、独立性和可重复性原则；单元测试的粒度一般是方法级别，最多也是类级别；核心业务、核心应用。核心模块的增量代码确保单元测试通过等。

那么接下来我们思考几个问题：

- 什么是单元测试？
- 为什么要编写单元测试？
- 什么是好的单元测试？
- 单元测试的常用框架有哪些？

这些都是本节将探讨的重点内容。



#### 2. 单元测试相关概念



##### 2.1 单元测试和集成测试

很多人一直在写单元测试，却不知道单元测试和集成测试的区别，认为：“使用能够编写单元测试的框架编写的测试就是单元测试”，这种认识是不全面的。

接下来，我们先了解单元测试和集成测试的概念和主要区别。

《单元测试的艺术》 第一章 单元测试的基础对集成测试和单元测试进行了描述 [2](https://www.imooc.com/read/55/article/1157#fn2)：

**单元测试**：

> 一个单元测试是一段代码，这段代码调用一个工作单元，并检验该工作单元的一个具体的最终结果。
>
> 如果关于这个最终结果的假设是错误的，单元测试就失败了。
>
> 一个单元测试的范围可以小到一个方法，大到一个类。

**集成测试**：

> “任何测试，如果它运行速度不快，结果不稳定，或者要用到被测试单元的一个或者多个真实依赖，我就认为它是集成测试。”
>
> 集成测试是对一个工作单元进行测试，这个测试对被测试的工作单元没有完全控制，并使用单元的一个或多个真实依赖物，例如时间、网络、数据库、线程或随机数产生器等。

**两者的主要联系和区别**：

集成测试和单元测试同样都很重要，都是验证系统功能的重要手段。

但是集成测试运行通常更慢，很难编写，很难做到自动化，需要配置，通常一次测试的东西过多。

集成测试会使用真实的依赖，而单元测试则把被测试的单元和其依赖隔离，以保证单元测试的高度稳定，还可以轻易控制和模拟被测试单元的行为方面。

因此**单元测试和集成测试最主要的区别之一就是测试中是否依赖 “真实环境”**。



##### 2.2 单元测试的重要性

很多人经常以 “时间紧，任务重” 或者 “单元测试没用” 为借口来拒绝编写单元测试。

但是 BUG 在软件的生命周期越早阶段发现，付出的代价越少。

单元测试可以让很多 BUG 在编码阶段就能够及时发现并解决，而不需要交给测试人员兜底，如果测试人员兜底失败，可能造成线上故障。

有了单元测试作保障，我们还可以放心对函数进行重构，如果重构代码导致单元测试运行失败，则说明重构的代码有问题。

**长远来看，单元测试对编码的益处（如提高代码质量和避免 BUG）远比编写单元测试的投入所花费的代价要大的多。**



##### 2.3 单元测试的方法

从我个人的理解来看，编写单元测试通常有两种方法，包括传统的单测方法和测试驱动开发，两种略有不同。

**单元测试的传统方法**大致流程如下：
![图片描述](img/5de47d44000145b503700438.png)

在国内，很多团队采用传统的方式，即先编写好代码，然后再编写单元测试来验证该段代码是否正确。

也有一些团队采用如下图所示的方式，即先编写单元测试，然后编写代码让通过测试。这种开发方式被称为**测试驱动开发（Test-Driven Development, TDD）**。
![图片描述](img/5de47be500018c2b07590326.png)

TDD 体现了 “以终为始” 的思想，即先制定目标，然后去验证是否实现了目标，而不是先做再去 “思考目标”。

**实践 TDD 的关键步骤**

1. 编写一个失败的测试来证明产品中代码和功能的缺陷；
2. 编写符合测试预期的产品代码，使测试通过；
3. 重构代码。

虽然 TDD 更 “出名”，具体采用哪种方法主要看团队约定和个人编程习惯。

由于团队没有强制约定，或者开发前参数不容易确定等原因，传统的单元测试方法依然被很多开发人员采用。



##### 2.4 何为优秀的单元测试？

既然要编写单元测试，那么好的单元测试标准是什么呢？

参考众多单元测试相关资料，我们得总结出优秀的单元测试应该具有以下几个特征：

> 满足功能：被检验的函数或类的逻辑行为满足预期功能；
>
> 满足 AIR 原则：单元测试应该可以自动执行；单元测试的用例之间要保持彼此独立；单元测试可以重复执行。
>
> 优秀的单元测试还应该具有编写容易，运行快速的特点。

在学习和开发过程中，看到很多人依然通过打印语句输出结果，通过 “肉眼” 来测试，这样如果对被测试的类或函数做出修改而无法满足功能要求，单元测试也会运行通过，就失去了单元测试的意义。

因此，建议大家在学习和工作开发过程中要遵循上述指导原则，编写出优秀的单元测试。



#### 3. Java 单元测试工具

**常用的 Java 单元测试有: JUnit、TestNG**。

TestNG 受 JUnit 和 NUnit 的启发，功能相似，但是比 JUnit 更强大。TestNG 不只为单元测试而设计，其框架的设计目标是支持单元测试、公共能测试，端对端测试，集成测试等。

JUnit 具体用法比较简单，如果想系统学习可[官方使用指南](https://junit.org/junit5/docs/current/user-guide/****)，参考《JUnit 实战 (第 2 版)》, TestNG 和 JUnit 非常相似，如果想深入学习，首推 [TestNG 官方文档](https://testng.org/doc/documentation-main.html)。

**主流的 Java mock 框架有: Mockito, JMockit, Easy Mock 。**

根据《What are the best mock framteworks for Java》一文 [3](https://www.imooc.com/read/55/article/1157#fn3)，我们可以看到三者的特点和优劣。

Mockito 简洁易用，有 PowerMock 拓展，允许静态函数测试，社区强大，对结果的验证和异常处理非常简洁、灵活。缺点是框架本身不支持 static 和 private 函数的 mock。

JMockit 简单易用；可以 mock “一切”，包括 final 类， final/private/static 函数，而其他 mock 框架往往只支持其中一部分；缺点是社区支持不够活跃，3 个 contributers 介乎只有一个在干活，学习曲线比较陡峭。

Easy Mock 上手简单，文档清晰；同样的社区较小，导致更多人选择其它的 mock 框架。

**还有很多其他配合单元测试的框架**，如强大的构造随机 Java 对象的 [Easy Random](https://github.com/j-easy/easy-random) ，构造随机字符串的 [Java Faker](https://github.com/DiUS/java-faker) 等。



#### 4. 总结

本节主要介绍了单元测试的概念、单元测试和集成测试的区别以及单元测试的必要性、主要步骤、主要框架等。

希望通过本节的学习，大家对单元测试能够有一个初步的理解。



### 单元测试构造数据的正确姿势



#### 1.前言

前面讲到了单元测试的概念和好处，讲到了 Java 单元测试常用框架。

写过很多单元测试的朋友会发现，单元测试的重要环节就是构造测试数据，单元测试构造测试数据往往非常耗时，这也是很多人不喜欢写单元测试的重要原因之一。

因此本节将重点讲述有哪些单元测试中构造数据的方式，各种构造测试数据方式的优劣以及实际开发中该如何选择。



#### 2. 构造单测数据的方式



##### 2.1 手动

所谓手动构造单元测试工具，是指在测试类或者函数中**直接声明测试数据，或在初始化函数中填充数据：**

```java
private List<String> mockStrList;

@Before
public void init() {
    mockStrList = new ArrayList<>();
    final int size = 10;
    for (int i = 0; i < size; i++) {
        mockStrList.add("something" + i);
    }
}
```

还可以在测试类中**编写私有 `mock` 数据的函数**来实现:

```java
private UserDO mockUserDO() {
    UserDO userDO = new UserDO();
    userDO.setId(0L);
    userDO.setName("测试");
    userDO.setAge(0);
    userDO.setNickName("test");
    userDO.setBirthDay(Date.from(Instant.now()));
    return userDO;
}
```

上述手动构造测试对象，当属性较多时，容易出错而且占据源码空间，而且不太优雅。



##### 2.2 半自动

当所要构造的数据为复杂对象(属性较多的对象)时，手动构造对象非常耗时，而且属性设置容易遗漏。

所谓半自动是指使用插件自动填充所要构造对象的属性。

比如m可以使用 **“Generate All setters”** IDEA 插件，选择 ”generate all setter with default value“ 填充默认值，效率提高很多。
![图片描述](img/5de710300001b4b305640229.png)

生成如下代码：

```java
private PdfData mockPdfData() {
    // 使用插件填充PdfData
    PdfData pdfData = new PdfData();
    pdfData.setId(0);
    pdfData.setName("some");
    pdfData.setWaterMark("test");
    pdfData.setPages(4);
    // 再次使用一次插件，填充PdfAttribute
    PdfAttribute pdfAttribute = new PdfAttribute();
    pdfAttribute.setWeight(1024L);
    pdfAttribute.setHeight(512L);
    pdfData.setPdfAttribute(pdfAttribute);
    return pdfData;
}
```

还有一种非常常见的”半自动“构造测试数据的方式，**采用 JSON 序列化和反序列化方式**。

将构造的对象通过 JSON 序列化到 JSON 文件里，使用时反序列化为Java对象即可：

```java
@Test
public void testPdfData() {
    // 构造测试数据
    PdfData pdfData = ResourceUtil.parseJson(PdfData.class, "/data/pdfData.json");
    System.out.println(JSON.toJSONString(pdfData));

    log.info("构造的数据:{}", JSON.toJSONString(pdfData));

    // 测试export 函数
    Boolean export = PdfUtil.export(pdfData);
    Assert.assertTrue(export);
}
```



##### 2.3自动

半自动的方式构造单元测试数据效率仍然不够高，而且缺乏灵活性，比如需要构造随机属性的对象，需要构造不同属性的 N 个对象，就会造成编码复杂度陡增。

因此， java-faker 和 easy-random 应运而生。



###### 2.3.1 java-faker

Java 构造测试数据中最常见的一种场景是：构造字符串。

如果想随机构造人名、地名、天气、学校、颜色、职业，甚至符合某正则表达式的字符串等，肿么办？

java-faker 是不二之选。

源码地址： https://github.com/DiUS/java-faker

基本用法如下：

```java
@Slf4j
public class FakeTest {

    @Test
    public void test() {
       // 指定语言
        Faker faker = new Faker(new Locale("zh-CN"));

        // 姓名
        String name = faker.name().fullName();
        log.info(name);
        String firstName = faker.name().firstName();
        String lastName = faker.name().lastName();
        log.info(lastName + firstName);
        // 街道
        String streetAddress = faker.address().streetAddress();
        log.info(streetAddress);

        // 颜色
        Color color = faker.color();
        log.info(color.name() + "-->" + color.hex());

        // 大学
        University university = faker.university();
        log.info(university.name() + "-->" + university.prefix()+":"+university.suffix());
    }
}
```

另外特别建议大家通过 Codota 的方式来查看其他开源项目中该类或者函数的用法：

![图片描述](img/5de710010001e64911360575.png)
还可以下载源码后查看核心类的核心函数来了解主要功能。

也可以通过源码提供的单元测试代码来学习更多用法，还可以通过调试来验证一些效果：
![图片描述](img/5de70f260001016209920533.png)



###### 2.3.2 easy-random

Java-faker 虽然可以构造不同种类的字符串测试数据，但是如果需要构造复杂对象就有些”力不从心“。

此时 easy-random 就要上场了。

源码地址：https://github.com/j-easy/easy-random

官方文档：https://github.com/j-easy/easy-random/wiki

easy-random 可以轻松构造复杂对象，支持定义对象中集合长度，字符串长度范围，生成集合等。

正如前面手动构造和半自动构造测试数据所给出的示例代码所示，构造复杂对象非常耗时且编码量较大，而使用easy-random，直接调用 `easyRandom#nextObject`一行代码即可自动构建测试对象：

```java
private EasyRandom easyRandom = new EasyRandom();

 @Test
 public void testPdfData() {
      // 构造测试数据
      PdfData pdfData = easyRandom.nextObject(PdfData.class);
      System.out.println(JSON.toJSONString(pdfData));

      log.info("构造的数据:{}", JSON.toJSONString(pdfData));

      // 测试export 函数
      Boolean export = PdfUtil.export(pdfData);
      Assert.assertTrue(export);
  }
```

Easy-random 还支持通过`EasyRandomParameters` 来定制构造对象的细节，如对象池大小、字符集、时间范围、日期范围、字符串长度范围、集合大小的范围等。

```java
EasyRandomParameters parameters = new EasyRandomParameters()
   .seed(123L)
  // 对象池大小
   .objectPoolSize(100)
  // 对象图的最大深度
   .randomizationDepth(3)
  // 字符集
   .charset(forName("UTF-8"))
  // 时间范围
   .timeRange(nine, five)
  // 日期范围
   .dateRange(today, tomorrow)
  // 字符串长度范围
   .stringLengthRange(5, 50)
  // 集合元素个数的范围
   .collectionSizeRange(1, 10)
  // 接口或抽象类时是否扫描具体的实现类
   .scanClasspathForConcreteTypes(true)
  // 是否重写默认的初始化方法
   .overrideDefaultInitialization(false)
  // 是否忽略错误
   .ignoreRandomizationErrors(true);

EasyRandom easyRandom = new EasyRandom(parameters);
```

建议大家一定要拉取 easy-random 源码，查看更多属性，包括 `EasyRandomParameters` 的默认值，以及运行其官方的单元测试来了解更多高级用法。

如可以查看其日期时间范围参数测试类: `DateTimeRangeParameterTests` ，来学习如何设置日期范围构造数据的日期范围：

```java
@Test
void testDateRange() {
    // Given
    LocalDate minDate = LocalDate.of(2016, 1, 1);
    LocalDate maxDate = LocalDate.of(2016, 1, 31);
    EasyRandomParameters parameters = new EasyRandomParameters().dateRange(minDate, maxDate);

    // When
    TimeBean timeBean = new EasyRandom(parameters).nextObject(TimeBean.class);

    // Then
   assertThat(timeBean.getLocalDate()).isAfterOrEqualTo(minDate).isBeforeOrEqualTo(maxDate);
}
```

我们不仅可以通过官方的单元测试来学习该框架的用法，还通过源码单元测试的范例来学习如何更好地编写单元测试。

可以在单元测试中打断点来观察构造对象的属性值，甚至可以通过单步来研究构造对象的过程。

更多高级用法，请自行拉取源码继续学习。



#### 3 如何选择？

前面讲到了构造单元测试数据的常用手段主要分为三种：**手动构造、半自动、自动构造**。

**那么该如何做出恰当的选择呢？**

下面给出一些建议：

- 当构造的测试数据非常简单时，如构造一个整型测试数据或者待构造的对象属性极少时，可以使用手动构造的方式，简单快速；
- 当待构造的对象属性均需要手动修改时，建议采用半自动的方式，使用插件构造测试对象并手动赋值或者使用JSON 反序列化的方式；
- 当待构造的测试数据为特定字符串时，如人名、地名、大学名称时，建议使用 java-faker；
- 当待构造的测试对象较为复杂时，如属性极多或者属性中又嵌套复杂对象时，建议使用 easy-random。



#### 4 总结

本小节主要介绍了**构造单元测试数据的几种常见手段**，如手动构造、半自动、自动三种方式。并介绍了**每种方式的常见构造方法以及各自的优劣**，并给出了如何**根据具体场景做出恰当的选择**。

希望大家在学习其他知识时，也要对知识进行归类和对比，这样才能深刻理解知识，才能举一反三。

下一节将给出单元测试的一些具体案例。



#### 5 课后作业

拉取 java-faker 和 easy-random的源码，运行关键类的单测来快速学习它们的用法。



### 单元测试之单测举例



#### 1.前言

前面我们讲到了构造单元测试数据的几种方式，接下来我们将讲述如何编写单元测试。

《手册》 第 29 页有对数据库单元测试的规定[1](https://www.imooc.com/read/55/article/1159#fn1)：

> 【推荐】和数据库相关的单元测试，可以设定自动回滚机制，不给数据库造成脏数据。或者
> 对单元测试产生的数据有明确的前后缀标识。

那么单元测试还有哪些注意事项，除了数据库相关的单元测试外，其它的单元测试又该如何去写呢？



#### 2.对哪些代码写单测？

实际开发中，主要对**数据访问层**、 **服务层**和**工具类**进行单元测试。

正如前言中所说，**数据库相关的单元测试**，一般要设置自动回滚。除此之外，还可以整合H2等内存数据库来对数据访问层代码进行测试。

**工具类的单元测试**也非常重要，因为工具类一般在服务内共用，如果有 BUG，影响面很大，很容易造成线上问题或故障。一般需要构造正常和边界值两种类型的用例，对工具类进行全面的测试，才可放心使用。此时结合注释小节所讲的内容，需将典型的调用和结果添加到注释上，方便函数的使用者。

**服务层的单元测试**，一般要依赖 mock 工具，将服务的所有依赖都 mock 掉。其本质是 “控制变量法”，将原本依赖的 N 个 “变量" 都变为“常量”，只观察所要测试的服务逻辑是否正确。



#### 3.单元测试的结构

大家一定要牢记编写单元测试的核心逻辑，其结构如下：
![图片描述](img/5de7121d0001cbeb04870121.png)

典型的单元测试可分为三个阶段，分别为**准备、执行和验证**[2](https://www.imooc.com/read/55/article/1159#fn2)。

**准备阶段（Given）** 主要负责创建测试数据、构造mock 方法的返回值，准备环节的编码是单元测试最复杂的部分。需要注意的是 Mockito 库中以 when 开头的函数其实是在准备阶段。

**执行阶段（When）** 一般只是调用测试的函数，此部分代码通常较短。

**验证阶段（Then）** 通常验证测试函数的执行的结果、 准备阶段 mock 函数的调用次数等是否符合预期。



#### 4.单元测试方法命名

早期必须在单元测试函数命名前加入 ‘test’ 前缀。现在已经不推荐这么使用，一般采用驼峰。

也会有很多人会将太多描述放到测试函数命名中，这也不太推荐，此种情况应该放到函数的注释中。

推荐的命名格式如：`shouldReturnItemNameInUpperCase()`。



#### 5.单元测试举例

数据访问层测试，只不过是将正常的环境加入了回滚或者采用内存/内嵌数据库，难度不大，这里就不给出具体范例。本文将重点讲述工具类的测试和服务层的测试。



##### 5.1 工具类的测试

学习工具类的单元测试，强烈推荐大家参考 [guava](https://github.com/google/guava)、[commons-lang3](https://github.com/apache/commons-lang)、 [commons-collection4](https://github.com/apache/commons-collections]) 这三个知名开源工具类项目的源码的单元测试代码。

如commons-lang3 包的 `StringUtils#contains` 源码:

```java
// Contains
//-----------------------------------------------------------------------
/**
 * <p>Checks if CharSequence contains a search character, handling {@code null}.
 * This method uses {@link String#indexOf(int)} if possible.</p>
 *
 * <p>A {@code null} or empty ("") CharSequence will return {@code false}.</p>
 *
 * <pre>
 * StringUtils.contains(null, *)    = false
 * StringUtils.contains("", *)      = false
 * StringUtils.contains("abc", 'a') = true
 * StringUtils.contains("abc", 'z') = false
 * </pre>
 *
 * @param seq  the CharSequence to check, may be null
 * @param searchChar  the character to find
 * @return true if the CharSequence contains the search character,
 *  false if not or {@code null} string input
 * @since 2.0
 * @since 3.0 Changed signature from contains(String, int) to contains(CharSequence, int)
 */
public static boolean contains(final CharSequence seq, final int searchChar) {
    if (isEmpty(seq)) {
        return false;
    }
    return CharSequenceUtils.indexOf(seq, searchChar, 0) >= 0;
}
```

对应的单元测试代码如下:

```java
@Test
public void testContains_Char() {
  // 不符合条件的特殊用例
    assertFalse(StringUtils.contains(null, ' '));
    assertFalse(StringUtils.contains("", ' '));
    assertFalse(StringUtils.contains("", null));
    assertFalse(StringUtils.contains(null, null));
  // 符合条件的用例
    assertTrue(StringUtils.contains("abc", 'a'));
    assertTrue(StringUtils.contains("abc", 'b'));
    assertTrue(StringUtils.contains("abc", 'c'));
  // 不符合条件的正常用例
    assertFalse(StringUtils.contains("abc", 'z'));
}
```

我们可看到，测试时除了选择符合条件的用例外，还要选择不符合条件的用例。其中不符合条件的用例可以还包括常规的用例和特殊用例（边界条件）。

再如 guava 的 `StopWatch#stop` ：

```java
/**
 * Stops the stopwatch. Future reads will return the fixed duration that had elapsed up to this
 * point.
 *
 * @return this {@code Stopwatch} instance
 * @throws IllegalStateException if the stopwatch is already stopped.
 */
@CanIgnoreReturnValue
public Stopwatch stop() {
  long tick = ticker.read();
  checkState(isRunning, "This stopwatch is already stopped.");
  isRunning = false;
  elapsedNanos += tick - startTick;
  return this;
}
```

根据源码我们可知，调用该函数后· `isRunning` 会被设置为`false`，如果重复调用会抛出 `IllegalStateException`，

因此，我们要测试已经停止后再次调用停止函数会的效果。

验证调用该函数后 `isRunning` 的确会被设置为`false`，如果重复调用会抛出 `IllegalStateException`，因此该函数的单元测试源码如下（注意该测试函数命名）：

```java
public void testStop_alreadyStopped() {
  stopwatch.start();
  stopwatch.stop();
  try {
    stopwatch.stop();
    fail();
  } catch (IllegalStateException expected) {
  }
  assertFalse(stopwatch.isRunning());
}
```



##### 5.2 服务层的测试

服务层的测试一般将底层的所有依赖都 mock 掉，最常用的框架为 Mockito、JMockit、 Easy Mock。

本小节的示例采用的是 Mockito。

核心场景如 ：A 类的某函数依赖 B 类的某函数和 C 类的某函数，而 B 类又依赖 E 类和 F 类，C 类又依赖 D 类，等等。
![图片描述](img/5de7124c0001328805010333.png)

如果要测试 A 类的某个函数，则需要 mock B类 和 C 类的对象。测试者可以指定 B 的某个函数接受某个参数返回固定的结果，指定C接受特定参数，返回特定结果，然后调用 A的对应函数，验证 A的返回值是否符合期待。
![图片描述](img/5de710e50001a46704940323.png)

注：为了简化，此处并没有采用标准的类图方式作图。

此时有些朋友可能会有一个疑问，**为什么不 mock D 、E 和 F 等其它类呢？**

其实这就是本专栏特别强调学习时要重视“是什么”的原因。单元测试从思想上来讲就是“控制变量法”，即将依赖变为“常量”，只有待测试的函数参数是“变量”，通过输入参数推测出结果，和实际的结果去对比，才可以更好地验证其正确性。

因此，我们只需要把它的直接依赖变成 “常量”即可，其它的依赖 mock 没有意义。

另外，大家一定要注意单元测试和集成测试的区别，不要将单元测试和集成测试混在一起。

下面给出一个简单示例：

待测试的服务接口：

```java
public interface ItemService {

    String getItemNameUpperCase(String itemId);
}
```

待测试的服务的实现类：

```java
@Service
public class ItemServiceImpl implements ItemService {

    @Resource
    private ItemRepository itemRepository;

    @Override
    public String getItemNameUpperCase(String itemId) {

        Item item = itemRepository.findById(itemId);

        if (item == null) {
            return null;
        }
        return item.getName().toUpperCase();
    }
}
```

可见该服务依赖数据访问组件 `ItemRepository`。

根据前面的单元测试的结构和命名建议，我们对该函数编写单元测试代码：

```java
import org.junit.Before;
import org.junit.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

public class ItemServiceTest {

    @Mock
    private ItemRepository itemRepository;

    @InjectMocks
    private ItemServiceImpl itemService;

    @Before
    public void setUp(){
        MockitoAnnotations.initMocks(this);
    }

    /**
     * 如果从存储层查询到一个Item, 那么它的 name 将被转化为大写.
     */
    @Test
    public void shouldReturnItemNameInUpperCase() {

        // Given
        Item mockedItem = new Item("it1", "Item 1", "This is item 1", 2000, true);
        when(itemRepository.findById("it1")).thenReturn(mockedItem);

        // When
        String result = itemService.getItemNameUpperCase("it1");

        // Then
        verify(itemRepository, times(1)).findById("it1");
        assertThat(result).isEqualTo("ITEM 1");
    }
}
```

测试函数采用驼峰命名并且体现出了该测试函数的核心含义。

可以看出在**准备阶段**，构造测试对象（数据）并 mock 掉底层依赖；**在执行阶段**直接调用待测试的函数；**在验证阶段** 对结果进行断言。

Mockito 的更多高级用法请参考[官方网站](https://site.mockito.org/)和[框架配套wiki](https://github.com/mockito/mockito/wiki)。如果需要 mock 静态方法、私有函数等，可以学习 [PowerMock](https://github.com/powermock/powermock/wiki/mockito)， 拉取其源码通过学习单元测试来快速掌握其用法。



#### 5.总结

本节主要给出单元测试在实际编程中的运用，给出了单元测试的结构、命名建议以及使用范例。希望大家在实际编程中能够举一反三，灵活运用，通过单元测试提高编码的质量。

下一节将给出 Java 学习宝典。



#### 6.课后作业

- 拉取 PowerMock 的源码，通过源码的单元测试来学习如何 mock 私有函数；
- 使用 easy-random 代替 shouldReturnItemNameInUpperCase() 函数构造测试数据部分的代码。



## 方法篇



### Java学习宝典



#### 1. 前言

前面章节围绕《手册》的一些重要知识点进行了学习和拓展。[1](https://www.imooc.com/read/55/article/1160#fn1)

**为什么有些人 “一年的经验用十年”，而有些人的学习和排错能力极强呢？**

因为很多人呆在舒适区不愿改变，不敢尝试新的方法，面对新的方法，工具本能地进行排斥，最终错过了快速成长的机会。

本节将系统阐述在几年的学习和工作中总结出来的 Java 学习的好方法，希望能够对大家有帮助。



#### 2. 学习的主要途径



##### 2.1 读书、看官方文档

学习的最常见手段之一就是读书。

就 Java 而言，正如前面章节研究问题时所提到的， Java 学习最核心的是 《Java 语言手册》、《JVM 语言手册》，因为这是官方，最权威的资料。

其次是围绕 java 语言典图书，如《Java 核心技术》、《Java 编程思想》、《深入理解 Java 虚拟机》、《码出高效》等。

还有 Java 开发岗位涉及到技术的官方手册，如 Spring、MyBatis、Hibernate 等官方手册等。

再如 Java 开发岗位涉及到的技术的经典图书，如《Redis 深度历险：核心原理与应用实践》、《深入理解 Apache Dubbo 与实战》等。

还有一些介绍编程技巧的书，如《重构：改善既有代码的设计》、《编写可读代码的艺术》、《代码整洁之道》、《修改软件的艺术》、《Effective Java》 等。

![图片描述](img/5deda6a90001fc4515141047.png)

如上图所示，越往底层信息密度越大，准确性越高，参考价值越大。

希望大家学习技术用法的同时，要注重源码、官方手册的学习，更加注重专业基础的学习。

**读书最关键的是选对书，其次是看书的方法。**

选取适合自己层次的图书很重要，初学者以介绍用法的图书为主，进阶的同学以读官方手册和相关的经典图书为主。

很多人读技术的书都会有这样的困惑：很多书读了一两遍还是会忘。

这是一个非常正常的现象。

为了克服遗忘，重点的图书需要反复阅读，这也就是所谓的 “书读百遍其义自见”，阅读时要加入一些思考，将所学知识和已经有的知识体系结合在一起。



##### 2.2 看视频

和读书类似，看视频也是 Java 学习的重要手段。

视频更加生动形象，尤其对于初学者入门来说帮助很大。

对于有一定基础，尤其是工作后的同学一般会因为视频讲解过慢，进阶的免费视频较少等原因，看视频学习的时间会减少。

**俗话说 “外行人看热闹，内行人看门道”。**

很多人尤其是新手看视频，仅仅学习视频讲解的内容本身，导致学习新的知识没有视频就不知道如何下手。

其实看视频学习不仅是看作者讲解的内容本身，更应该关注作者的编码风格，作者学习新知识的方式，作者的编码思路，这些才是更有价值的通用的技能，也是看视频学习的精髓所在。



##### 2.3 读源码

读源码是学习进阶的必由之路，但是同样是读源码，不同的方法效果差异很大。

对于 Java 程序员而言，最重要的就是读 JDK 的源码，其次是读一些经典框架的源码，如 Spring、 Dubbo 等。

通过读源码的注释，可以深入了解函数的主要功能，甚至核心思路。

通过阅读源码，可以了解到一些优雅的编码风格，设计思想。还可以通过源码来学习和理解其中所运用的设计模式。

读源码时要特别注重思考为什么要这么设计，不这么设计会有什么问题。

然而，很多 Java 初学者，甚至工作一两年的 Java 程序员，读源码抓不住重点，以记忆为主，而不是思考为主。

读源码时迷失在细节之中，而不能先整体后局部，从设计者的角度来读源码。

如何更好地阅读源码，在后面的章节中将会重点介绍。



##### 2.4 调试

IDE 的代码调试器也是 Java 学习重要手段。

通过代码调试可以清楚代码的运行轨迹，可以清楚地观察各个对象的状态。

但是很多 Java 初学者，甚至工作一两年的 Java 程序员， 也只是停留在打个断点，单步调试，并没有熟练掌握更高级的调试方法。

在后面的章节会系统地介绍调试的正确姿势。



##### 2.5 看专栏

现在博客的质量参差不齐，尤其中文技术博客各种相互抄袭，很多不错的技术博客容易断更等。

随着近些年知识付费的普及，专栏成为学习知识的一个重要途径。专栏都是围绕着一个主题写作，购买和学习专栏可以对某一块知识有一个全面和深入地理解。

建议大家可以通过购买专栏的方式，系统地将自己不足的模块补齐，而不是低效地碎片化学习。



##### 2.6 看公众号、博客等

对于 Java 程序员而言，技术公众号也是学习的重要途径。

比较不错的公众号有： 架构师之路、IT 技术精选文摘、JavaGuide、程序员小灰等等。

看一些分享学习和工作中的经验技巧的博客，也是学习的一个不错途径。

但由于公众号和博客的质量参差不齐，读公众号和博客要抱着怀疑的态度，不要轻信。

公众号和博客只能作为参考，不应该作为权威的依据。



##### 2.7 各种图

作为 Java 程序员，思维导图和 UML 图都是学习和设计的强有力工具。

通过思维导图，可以整理需求，梳理所学知识并构建知识体系。

UML 图更是需求分析、系统设计、梳理系统逻辑必不可缺的强大工具。

思维导图和 UML 图在后续的章节中也会给出全面的讲解。



##### 2.8 其他

还可以通过抓包（tcpdump -A | grep xxx）、反汇编、反编译，来学习 Java 相关知识。

也可以通过 Google 和 Stack OverFlow 来搜索问问题的答案。

不推荐使用百度搜索问题，是因为百度出的很多问题的答案千篇一律。

搜索到的解决方案也往往没有给出该问题最本质的原因。

而 Google 和 Stack OverFlow 搜索出来的资料质量相对较高，而且往往会给出问题的根本原因以及参考资料。



#### 3. 学习的方法



##### 3.1 推演验证

对于大多数工作的朋友来说，入职的部门一般都会有已经设计好的项目模块，这是我们学习进阶的**绝佳素材**。

但是很多人并不会重视和分析之前的设计，也体会不到这种学习方式给自己能够带来的极大成长。

我们可以使用推演验证的方式来快速掌握系统的设计、熟悉功能、学习设计思路等。而不是进入侧重记忆，导致容易遗忘，无法灵活运用的怪圈。

> 所谓的**推演验证**，就是当我们熟悉一块功能，或者我们想通过某个模块来提高编码和设计能力时，我们可以找到需求文档和已经上线的产品去对比使用，然后模拟自己出一个技术方案，包括数据库表的设计，分布式中间件的使用还有一些其它细节等。

我们通过需求文档、交互稿或者体验真实的功能后去反推实现方式，如果当初有技术文档，要和当初的实际的技术方案进行对比。通过对比找出自己设计的缺陷，思考对方当初为何这么设计，并努力找出对方的设计可以改进的地方。通过不断的推演和验证，自己的业务设计能力会提高的很快。

这种方法有点像学生时代做 “模拟题”，当初某个项目的设计方案就是我们 “模拟题的答案”，通过这种方式的训练，我们的 “工作经验” 会提高很快。而现实生活中，往往是我们没有和 “真题” 同等难度，甚至更难的 “模拟题” 的经验，而只是从自己趟过的坑，做过的项目，通过别人的指点来学习，这样就会收效甚微。

下面结合一个场景来为大家解释具体的做法：

> 比如很多人会发现市面上很少有很通俗易懂地教你如何根据实际的场景去设计数据库表结构的资料，肿么办？
>
> 对于还没工作的人来说，可以找一两个知名的开源项目；对于已经工作的人来说，可以通过自己开发的项目来学习表的设计。
>
> 如何学习呢？
>
> 我们先找某一个自己感兴趣的功能点，根据需求文档、页面表现等，熟悉功能。
>
> 然后根据功能自行**推演**，如应该会有几张表，每张表应该包含哪些字段等等。
>
> 然后去和实际的表结构去对比**验证**，如果不同，自己设计的表是否满足需求？对方的设计更好吗？好在哪里？自己是否有遗漏？等等。
>
> 通过多次训练，你会发现自己对表的设计掌握的会越来越好，你将更清楚为什么要这么设计以及如何设计。

后续的源码学习章节的其中一种高效的读源码方法也将采用 “推演和验证” 的方法。

只有真正尝试过这种方法的人才能体会到它的巨大价值，希望大家在平时开发和学习中多去尝试。



##### 3.2 教是最好的学

我们学了好多年，但是很多人从来不会主动探索高效的学习方法，在学习过程中见到的最好的学习方法之一就是 “教学相长”。

俗话说 “教是最好的学”，在中国的古代文献中有类似的说法。

《礼记・学记》：“学然后知不足，教然后知困。知不足，然后能自反也；知困，然后能自强也。故曰：教学相长也。

《兑命》曰：“‘学学半。’其此之谓乎。” 郑玄注：“学则睹己行之所短，教则见己道之所未达”。

另外著名的费曼学习法也是强调 “教学相长”[2](https://www.imooc.com/read/55/article/1160#fn2)：

> **费曼学习法**的灵感源于诺贝尔物理奖获得者理查德・费曼（Richard Feynman），运用费曼技巧可以深入理解知识点，并且记忆深刻不易遗忘。知识有两种类型，我们绝大多数人关注的是错误的那种。第一类知识关注了解某个事物的名称。第二类知识关注了解某个事物，这是两码事，通过费曼学习法可以让我们对事物的理解更透彻。

费曼学习法的四个步骤：

- 第一步：将学的知识教给别人；
- 第二步：回顾。遇到卡壳的地方，回顾原材料，重新学习；
- 第三步：将语言条理化，简化。只有能够用简单易懂的语言描述某个知识，才代表真正理解了某个知识；
- 第四步：传授。确保自己理解无误的情况下，将知识传授给他人。

因此我们学习一个知识点时可以尝试教给别人，如果没有合适的倾听者可以讲给 “小黄鸭”（虚拟的听众）听。

另外也可以将所需的知识写到博客中，也可以在技术群里和别人交流，这也是另外一种 “教”。



##### 3.3 PDCA 循环

PDCA 循环的来源和定义最早是由美国质量管理专家戴明提出来的，也称 “戴明环”。
![图片描述](img/5deda6890001932e05090486.png)

图：PDCA 循环示意图（图片来自百度百科）

核心内容如下：

1. **P（Plan）** 即计划。“凡事预则立，不立则废”，做事之前一定要有计划；
2. **D（Do）** 即所谓的执行。很多人有了计划没有执行力，很难有效果；
3. **C（Check）** 即检查。分析执行的结果，明确执行中存在哪些问题；
4. **A（Action）** 即处理。对成功的经验要保持，将其标准化；对失败的经验要总结，引起重视。

请注意，这里并不是为了介绍某个高大上的概念，而是希望大家能够真正去思考去理解。

为了更好地理解 PDCA 的价值，我将生活和开发中的一些场景进行描述以帮助大家理解。

> 在学生时代很多人考试之前会做模拟题或者历年真题，你会发现，很多人做完题后只是把答案抄上去再也不看了。导致的一个严重的问题是，做过的题大概率还错。

这其实就是走了弯路而不自知，浪费了时间却没有成效。

在我们学习 Java 和参与工作中，很多人一样会存在这种问题：

> 开发一个模块之前没有充分分析（甚至轻视）需求，着急编写代码，编写代码过程中不能够有意识的和 master 分支进行比对，代码部署到测试服务器时随便点点就算，如果上线后遇到 BUG 就进行修复。

很多人学习时更喜欢 “做更多试卷” 给自己带来的虚假成就感，而不是珍惜错题给自己带来的价值。

同样地，做项目时，很多人喜欢做更多项目给自己带来的虚假 “成长”，每个项目并没有输出对下一个项目有用的经验。

这也是很多人 “一年经验用了十年” ，成长很慢的原因之一。

**那么该怎么做呢？**

编写代码之前一定要充分了解需求，做好技术方案，然后再去编码设计。

> 很多有经验的人都会认为，其实软件开发最关键的是需求的分析，技术方案的设计，而编码只不过是一个时间问题。

编写完代码后将自己的分支和 master 比对，检查是否有冲突，进行自我代码审查。

> 养成和 master 对比代码的习惯，你可以在测试之前就发现自己编码的问题。

每个项目上线后，总结这些项目学到的经验，存在哪些问题，如果有 BUG，分析 BUG 的原因。

> 比如
>
> [1] 这次测试过程中发现某个错误日志在测试环境就已经看到过，但是认为和本功能无关就没注意，最终发现是本功能的修改触发了其他功能的 BUG。
>
> 此时我们可以将 “测试时一定要重视警告和错误日志” 积累到我们的验证流程中；比如测试时可以养成好的习惯，使用 tail -f error.log 看看有没有错误日志。
>
> [2] 这次开发过程中，我们本地代码都没编译通过就提交到了 git 仓库并在测试服务器打包部署，导致部署失败。
>
> 此时，我们可以将 “提交之前一定要本地编译通过” 积累到我们的开发经验中。比如 maven 项目我们可以每次提交前执行 mvn clean compile。
>
> [3] 比如我们代码上线后某个地方忘了打日志，不方便我们分析排查问题。那么我们可以梳理出哪些地方该打日志，下次一定开发时要重视起来。
>
> 等等。

然后将经验梳理到云笔记中或者思维导图中，并让它们成为我们的习惯，不断地帮助下一次开发。

这样才能造成良性循环，这样经验才能有目的的快速累积。

并不只是学习新的技术才叫学习，决定一个人是否牛的标准之一是他能否在某一个领域超越绝大多数人。

**很多人学习进阶的速度慢，不仅仅在于有些东西没有学，而是不懂学习的方法，不懂总结和反思**。

孔子说 **“温故而知新，可以为师矣”** 诚如是。



#### 4. 总结

本节主要讲述了 Java 学习的主要途径，包括读书、看视频、读源码、调试等；还推荐了几种高效的 Java 学习方法，如费曼学习法、PDCA 循环等。本小节还分析了很多人学习和进阶速率慢的主要原因，即不重视方法，不重视总结和反思。人最可怕的是一直呆在舒适区，不愿意改变，不愿意接受新鲜事物，不愿意学习新的方法。希望通过本节的介绍，对大家能够有所启发。



### 代码调试的正确姿势



#### 1. 前言

《手册》对代码调试介绍较少，其中第 37 页 SQL 语句章节有如下描述 [1](https://www.imooc.com/read/55/article/1161#fn1)：

> 【强制】禁止使用存储过程，存储过程难以调试和扩展，更没有移植性。

由此可见，可调试也是 Java 程序员编码要考虑的重要一环。

可以说，代码调试是 java 程序员的必备技能之一。

但是很多 Java 初学者和工作一两年的程序员仍然存在使用 “打印语句” 来代替调试的现象。还有很多 Java 程序员只了解最基本的调试方法，并没有主动学习和掌握高级的调试技巧。

本节将在 IDEA 中展示常见的调试方法和高级的调试技巧。



#### 2. 调试的好处

调试和日志是排查问题的两个主要手段。

如果没有调试功能，很多问题的排查更多地将依赖日志。

但是日志无法直观地了解代码运行的状态，无法实时地观察待调试对象的各种属性值等。

调试工具非常强大，很多调试器支持 “回退”，自定义表达式，远程调试等功能，对我们的学习和排查问题有很多帮助。



#### 3. 调试的基本方法

调试的基本步骤：

1. 设置断点
2. 调试模式运行
3. 单步调试

如图所示，在单元测试类的 33 行设置断点，然后在测试类或函数上执行 debug， 则程序执行到断点时会暂停。
![图片描述](img/5dedaa540001bd8209270571.png)
此时可以看到所有的变量：

![图片描述](img/5dedaa2e000117e410090592.png)
常见的调试功能按钮如上图所示。

1 表示 Step Over 即跳过，执行到下一行；

2 表示 Step Into 即步入，可以进入自定义的函数；

3 表示 Force Step Into 即强制进入，可以进入到任何方法（包括第三方库或 JDK 源码）；

4 表示 Step Out 即跳出，如果当前调试的方法没问题，可以使用此功能跳出当前函数；

5 表示 Drop frame 即移除帧，相当于回退到上一级；

6 表示 Run to Cursor 即执行到鼠标所在的代码行数。

其中 1、2、3、4、6 这 5 个功能，以及 “variables（变量区）” 初学者用的最多。

通常设置断点后，通过单步观察运行步骤，通过变量区观察 “当前” 的数据状况，来学习源码或者排查错误的原因。



#### 4. 调试的高级技巧



##### 4.1 多线程调试

设置断点时，在断点上右键可以选择断点的模式，选择 "Thread" 模式，可以开启多线程调试。

![图片描述](img/5dedaa150001d0f509350358.png)
可以将一个线程断下来，通过 “Frames” 选项卡切换到不同线程线程，控制不同线程的运行。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

该调试技巧在模拟线程安全问题时非常方便。



##### 4.2 条件断点

和多线程调试类似，我们还可以对断点设置条件，只有满足设置的条件才会生效。

![图片描述](img/5deda9e40001e40d12400451.png)
该功能在测试环境中非常有用。

比如你提供视频的转码功能作为二方库给其他团队使用，此时代码发布到测试环境，如果设置普通断点，那么所有的请求都会被暂停，影响其他功能的调试。

此时就可以设置条件断点，将某个待测试的视频 ID 或者业务方 ID 等关键标识作为断点的条件，就不会相互影响。

**如果我们想对某个成员变量修改的地方打断点，但是修改的地方特别多怎么办？**

难道每个地方都要打断点？

如下图所示，我们可以在属性上加断点，选择在属性访问或修改时断点，还可以加上断点生效的条件。

![图片描述](img/5deda9ce0001faa708540476.png)



##### 4.3 “后悔药”

在基本调试方法部分讲到，按钮 5 表示 Drop frame 即移除帧，相当于回退到上一级，这给我们提供了 “后悔药”。

当我们调试某个问题时，一不小心走过了，往往会重新运行调试，非常浪费时间，此时可以通过该功能实现 “回退”。

比如我们在 33 行设置断点，通过 step over 走到了 第 36 行。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)
然后我们通过 step into 来走到了 `ItemServiceImpl` 的 第 16 行，如果我们想回退到上一层，直接使用 drop frame 功能即可回退到上图状态重新调试。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 4.4 “偷天换日”

我们实际调试代码时，会有这样的场景，调用的参数传错了。修改参数重新运行？

不需要，我们可以在调试过程中对调试对象的值进行动态修改。

如程序运行到 39 行时 result 的值为 “ITEM 1”，如果我们想对其进行修改。

![图片描述](img/5deda96800010ba307530722.png)

此时在 variables 选项卡中选中 `result` 变量，然后右键，选择 “set value” 菜单，即可对变量的值进行修改。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)
修改后可以继续调试观察运行结果。



##### 4.5 表达式

在调试过程中可以对变量执行表达式，这对排查问题有很大帮助。

![图片描述](img/5deda9370001b10a09660583.png)
如图所示，我们可以对 `mockedItem` 变量执行表达式并查看结果：

![图片描述](img/5deda85700011c9b07620429.png)

比如我想查看 spring 的上下文的 `beanFactory` 中是否有名为 `demoController` 的 bean 定义映射，可以使用功能该功能查看：

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

我们还可以通过表达式为待观察的集合添加数据：
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

大家可以根据实际的情况，灵活运用。



##### 4.6 watch

如果我们想在调试过程中查看某个对象的某个属性，总是使用表达式很不方便，是否可以将表达式计算的结果总是显示在变量区域呢？

答案是有的，使用 watch 功能即可实现。

在变量区右键 ->"New Watch"–> 输入想要观察的表达式即可。

如下图所示，我们可以输入 “order.getOrderNo ()” ，这样就不需要调试时总展开 Order 对象来查看订单编号了。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

这里只是举一个非常简单的例子，该功能如果能根据实际的场景灵活使用非常方便。



##### 4.7 看内存对象

比如我们想通过代码调试来研究下面的示例代码共产生了几个值为 150 的对象：

```java
public static void main(String[] args) {
    Integer c = 150;
    System.out.println(c==150);
}
```

我们可以在 Memory 选项栏下，搜索 Integer 就可以看到该类对象的数量，双击就可以通过表达式来过滤，非常强大。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 4.8 异常断点

有些朋友可能遇到过这种问题，在一个循环中有一个数据报错，想在报错的时候断点，无法使用条件断点，而且循环次数很多，一次一次断掉放过非常麻烦。肿么办？囧…

情况下面的示例代码：

```java
public static void main(String[] args) {
    for (int i = 0; i < 100; i++) {
        some(i);
    }
}

private static void some(int i) {
    if (RandomUtils.nextBoolean()) {
        throw new IllegalArgumentException("错了");
    }
}
```

通常这种情况我们是知道异常的类型的。

第一步，在我们想要研究的地方断点，比如我们想研究 i 为几时，条件为 true（只是一个演示）。我们先在 14 行断点；

第二步：我们可以点击左下角的红色断点标记，打开断点设置界面；

第三步：点击左上角的 + 号，添加 “Java Excepiton Breakpoints” 将 `IllegalArgumentException` 添加进去；

第四步：切换到我们的断点处，即下图所示的 DebugDemo.java:14 处，在 “Disable until breakpoint is hit” 处选择该异常。
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

此时再执行断点调试，即可捕捉到发生异常的那次调用。

同样地我们也可以通过调用栈查看整个调用过程，还可以通过移除 frame 来回退到上一层。

此外，我们还可以使用 arthas 的 watch 功能查看异常的信息。



##### 4.9 远程调试

现在大多数公司的测试环境都会配置支持远程调试。

远程调试要求本地代码和远程服务器的代码一致，如果使用 git ，切换到同一个分支的同一次提交即可。

设置虚拟机参数：

> -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000
>
> -Xdebug

在 IDEA 中 运行和调试配置中，设置 remote 的 host 和 port 即可。

![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)



##### 4.10 其它

IDEA 的调试器非常强大，还支持调试时主动抛出异常，强制退出等功能：
![图片描述](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

希望大家在平时调试代码时，可以尝试更多新的技巧，节省时间，快速定位问题。



#### 5. 总结

本节主要介绍了代码调试的常见用法和高级功能，掌握好调试技巧将极大提高我们排查问题的效率。

真正开发时往往是多种调试方法结合在一起，比如可以将修改变量值和回退一起使用，也可以将条件断点和修改变量值，单步等功能一起使用。

掌握好调试技巧，对快速定位问题，学习源码等都有很大的促进作用。

当然，代码调试还有很多其他的高级调试技巧可查看 IDEA 官方文档学习，也可以在开发中自行探索。



#### 6. 课后题

课后大家编写测试代码自行练习：回退、修改变量值的功能。



### 阅读源码的正确姿势



#### 1. 前言

Java 学习和进阶离不开阅读源码，但是很多人只知道阅读源码却不知道如何阅读源码更有效。

很多人面对源码无从下手，也有很多人阅读源码刚开始就陷入细节，看着看着就晕了，很难坚持下去。

也有很多人看了很多源码，最终都 “忘了”，没留下什么印象。

我自己也遇到过类似的问题，通过探索和交流总结了一些经验，在此分享给大家。



#### 2. 读源码究竟读什么？

很多人只是知道阅读源码是进阶的一个重要步骤，但是在阅读之前并不是很清楚到底要通过源码学到什么。

如果读源码之前想不清楚这件事，很容易 “走马观花”，收获无多。

通过阅读源码可以学习到很多知识，如：编码规范，包括类、函数、属性的命名，注释的规范等；优秀程序员的编程思想；学习一些高级的编程技巧；某些功能或特性的核心原理；可以学习到一些好的设计原则、设计模式如何落地。



#### 3. 阅读源码的思路

阅读源码的方法和心态很重要，很多人想一口气吃个大胖子，急于求成最后适得其反。

很多人急躁的心情是可以理解的，想早点攻克某个框架源码，但是大家可以回想一下打游戏的场景，想打好游戏，通常需要学习各种通关技巧，需要先 “打野”。

下面介绍几个阅读源码的思路。



##### 3.1 从设计者的角度看源码

**从设计者的角度看源码是最有效的方式**。

源码也是人写出来的，源码的作者编写代码之前也是在头脑中思考过的。

源码，尤其是复杂源码，都是符合 “任务拆分” 的原则的，即一个大的功能分为几个核心的步骤，分别编写代码。

这也符合罗伯特・C・马丁（Robert Cecil Martin）所提出的面向对象五大基本原则之一的：单一职责原则。

> 单一职责原则：一个类或者模块应该有且只有一个改变的原因

因此我们学习源码要想好编写这个功能应该有哪些步骤，再去和源码对比。

这样才能验证自己思考问题的角度是否正确，是否有遗漏。

通过对比能够清楚地知道作者为什么要这么设计，作者的源码比自己所设想的好在哪里，这样才不容易遗忘。

**这就像学生时代做数学题一样，很多人会发现如果我们不做题就直接看答案，我们会认为问题都很简单，自己都会。但是真正脱离答案去做题时，往往并不会做。这也像我们拿着复杂迷宫的答案图纸去看迷宫时，会认为迷宫并不难，但是没有提前看答案时，破解迷宫的难度是要大很多的。**

下面举一个非常简单的例子：

在开发时，需要借助 okhttp 封装一个 HTTP 请求工具类，其中涉及到编写一个判断请求是否成功的函数。

正如很多人认为地那样，在封装地函数中直接判断响应码是否等于 200 即可。

```java
public boolean isSuccessful(Integer code) {
    return 200 == code;
}
```

但是当我们去查看 `okhttp3.Response` 的 `isSuccessful` 的写法：

```kotlin
/**
 * Returns true if the code is in [200..300), which means the request was successfully received,
 * understood, and accepted.
 */
val isSuccessful: Boolean
  get() = code in 200..299
```

突然发现我们的想法不够严谨，响应码从 200 到 299 都应该算请求成功。

这样我们对源码的某个细节的印象就会非常深刻，更加清晰地了解到自己思路的不严谨性。

如果我们的 “猜想” 核心的步骤和最终和源码比对，如果和作者的逻辑非常一致时，我们就很开心，这也是看源码的乐趣之一。如果不一致，通过对比完善自己的思路。

**通过这种方式去读源码能够不断纠正我们的思路，不断发现我们的问题，这是阅读源码非常重要的一个目的。**



##### 3.2 先整体后局部

俗话说 “磨刀不误砍柴工”，这几乎是尽人皆知的道理，但是学习编程时很多人依然会着急看源码，不重视背景知识，不重视框架的整体思想，导致后面浪费更多地时间。

**为了避免过早陷于局部而缺乏全局观念，应该先从整体了解一个技术的核心模块再去学习每个具体模块的源码。**

比如我们学习 dubbo 源码之前必须想了解该框架的主要模块以及之间的关系。

[dubbo 架构图](https://dubbo.apache.org/zh-cn/docs/user/preface/architecture.html)：

![图片描述](img/5dedbf390001d5ea05210349.png)

先了解架构的核心角色以及调用关系，再去学习源码会更容易一些。

**先仔细阅读官方手册再去学习源码**，很多人不重视官方手册，学习很久甚至工作很久，连核心技术栈的官方手册都没认真看过一遍，这是一件非常可怕的事情。

对要学习的技术有一个整体地了解之后，可以去拉取源码，去看源码包含哪些模块，每个模块的大体功能是什么，各个模块之间有什么关系等，然后再去看代码的细节。而大多数人读源码，会认为这些不重要，会急于读源码，导致效果不好。



##### 3.3 由易到难

尤其是对于很多新手来说，连核心技术栈使用都不熟悉的情况下，直接看其源码很容易遭受很大打击。

因此要根据自己的阶段去选择适合自己的框架来阅读。

这一点和打游戏是非常一致的，一般开局都先 “打野”，通过 “打野” 来提升等级获取装备等，再去和高级的敌人对抗。

对于初学者而言，可以先从开发中常用的简单的框架入手，如 [commons-lang ](https://github.com/apache/commons-lang)、[commons-collections](https://github.com/apache/commons-collections)、 [guava](https://github.com/google/guava) 等。从看这些简单的源码积累经验，然后再去学习 spring 、spring boot、dubbo 等框架的源码。

另外要先保证能够熟练使用，再去学习源码效果会更好一些。如果连使用都不会就直接去学习源码，是一种非常不理智的行为。

另外学习从来不是匀速的，大家也明白 “欲速则不达” 的道理，建议可以先从简单的框架入手，积累经验后快速将这种学习的能力迁移到自己想研究的框架中去。



##### 3.4 带着问题看源码



###### 3.4.1 通用的问题

看源码和学某个技术之前，要重点思考几个能从整体理解该项目的问题：

- 这个项目主要核心功能是什么？
- 这个框架能解决什么问题？
- 有没有同类的框架，有啥异同？

很多人会忽略这些问题，认为这些问题不重要，导致虽然能用起来，却对框架的使用场景、解决的本质问题都不清楚。



###### 3.4.2 工作中遇到的问题

对于很多人而言，大多数时间都花费在工作上，业余时间并没那么多，那么如何去学源码呢？

其实未必需要有大量完整地时间才可以去学习源码，**我们在开发过程中遇到问题时可以顺便进入源码来研究问题**。

在工作任务不是特别忙的时候，可以通过自己项目引用的 jar 包进入源码中看一下平时常用的注解是如何解析的，常用的函数具体实现是怎样的，常用的类中还有哪些其它函数等。

当学习和工作中遇到问题时，如果能够借机深挖，就可以借助这个问题带动源码的部分内容的学习。

如在 《虚拟机退出时机问题研究》小节所举的例子，下面代码打印语句还没来得及执行就结束了：

```java
public class CompletableFutureDemo {
  public static void main(String[] args) {
    CompletableFuture.runAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2L);
        } catch (InterruptedException ignore) {
        }
        System.out.println("异步任务");
    });
 }
}
```

最终我们通过进入 `runAsync` 函数的源码跟踪到 `ForkJoinPool#registerWorker` 函数发现， `ForkJoinPool` 的工作线程类型为守护者线程。

我们就借着这个机会，学习了 `CompletableFuture` 源码的部分知识点。

下面介绍另外一个问题，比如某同学使用 MyBatis 时，运行项目测试时发现找不到自定定义好的 Mapper，咋回事呢？

此时你要想出各种可能性：

- 包名是不是写错了扫描不到？
- 接口是不是写成类了？

等等最有可能的问题，然后依次排查。

最终发现是因为自己误将 Mapper 接口定义为类导致的，将类 (class) 改成接口 (interface) 就好了。

那么为啥会这样呢？大家千万不要就此打住。

了解 MyBatis 会通过 MapperRegistry 来注册和获取 Mapper 对象的代理，我们进入添加 mapper 的核心代码：
![图片描述](img/5dedbf190001279f13250770.jpeg)

可以看到该函数会先判断 Mapper 是否为接口类型，如果是接口类型才会注册此映射的代理对象，因此问题就非常明确了。

每一个问题都是我们学习源码的好机会，希望大家能够有这种意识。

很多人恰恰是平时不用心，临近找工作突击，才导致学啥都不深入，结果可想而知。

随着遇到的问题越来越多，看过常见的类的源码的函数越来越多，对源码的理解就越来越深刻，越来越熟悉。



###### 3.4.3 看 issues

通过看开源项目的 issues， 你可以发现该项目的潜在 BUG。

可以了解同一个问题，不同人的解决思路，以及官方最终采用的是哪种方案。

很多人的解决方案，对你实际的业务开发也有很大帮助。



###### 3.4.4 看时序图

有些人可能会想到，可以根据源码画出时序图来理解源码，但是画图非常耗时，怎么办呢？

IDEA 插件 [SequenceDiagram](https://github.com/Vanco/SequencePlugin) 就派上用场了，这个插件非常赞，可以根据源码绘制出调用时序图，对学习源码帮助极大。

![图片描述](img/5dedbf0700014fb216000894.jpg)



###### 3.4.5 看错误堆栈信息

程序运行出错时，是我们学习的最佳时机。

大家可以通过 [Stack trace to UML 插件](Stack trace to UML ) 绘制出出错的调用时序图，了解调用的顺序。

如下面出错信息：

![图片描述](img/5dedbee80001268e14820422.png)

安装好插件后，通过菜单：Analyze > Open Stack trace to UML plugin + Generate UML diagrams from stacktrace from debug ，将绘制出下面的时序图：

![图片描述](img/5dedbed7000174d615380714.png)
当异常堆栈信息非常多时，通过该插件绘制出的时序图将非常有助于帮助我们了解调用链，理解源码。



##### 3.5 带着场景学源码

比如从设计模式的角度去学习源码。

可以从设计模式的六大原则来思考源码的设计，思考源码是如何体现这几种原则的。

> 设计模式六大原则：单一职责原则、里氏替换原则、依赖倒置原则、接口隔离原则、迪米特法则、开放封闭原则。

还可以结合《设计模式之禅》这本书或者菜鸟教程中设计模式的教程，了解具体某些设计模式的特点、使用场景、优点、缺点等。

然后从 JDK 或者 Spring 等自己想学习的框架源码中去寻找这些设计模式的身影。

通过这种方式可以更清楚设计模式该如何落地，从更多角度去了解源码。

这里不详细展开，希望大家自行学习。



##### 3.6 通过源码的单元测试来学习源码

正如前面一些章节所提到的，大多数知名的 Java 开源项目都会有非常完善的单元测试，这是我们学习源码的一个非常重要的突破口。我们可以运行单元测试来调试源码，熟悉核心类的功能。

![图片描述](img/5dedbebb0001df2512060729.png)
可以直接根据类名搜索，也可以通过找到该类，使用 “find usages” 功能来找到其单元测试代码。



##### 3.7 通过 DEMO 学源码

大家可以使用官方的例子或者自己写例子运行，来体会某个项目的用法，研究其特性。

在这里推荐一个高质量的英文技术文章网站 [baeldung](https://www.baeldung.com/category/java/), 几乎所有的文章都有 [配套代码](https://github.com/eugenp/tutorials) , 我们可以直接通过该网站的代码运行学习某些知识点，某些框架。

![图片描述](img/5dedbe7d0001749904810773.png)

大家学习某个框架，还可以自行去 github 找到相关的范例，运行学习。

另外，超级推荐大家通过自己开发的项目来学习 Spring 源码。大家可以对照着官方文档、对照着 Spring 的源码教程等，观察自己项目中某个 Spring 类的使用，还可以在项目测试时偶尔进到源码中断点，通过调试自己的项目来学习源码。



#### 4. 阅读源码的技巧



##### 4.1 实现 “简易版” 是学习的重要途径

比如学习 Spring 源码之前，可以根据自己平时使用 Spring 的方式，自己实现简易版的 Spring，记录自己编写代码的核心步骤，以及核心步骤的缺点和遇到的问题。待真正去阅读源码时，很多问题豁然开朗。

可能很多人会认为，不是所有的代码都有简易版。的确如此，但是只要思维灵活，方法总比困难多。

如可以购买或者寻找《Spring5 核心原理与 30 个类手写实战》书本所配套的[简单版 Spring 代码](https://github.com/gupaoedu-tom/spring5-samples)，并且自己尝试从最简单版去改编，梳理清楚核心逻辑，并记录这个过程中遇到的困难。再去读 Spring 源码就会容易很多。

比如想读 dubbo 源码，可以在 github 上找一些简单的 Java RPC 框架看会后再去看 dubbo 的源码。



##### 4.2 寻找程序入口是一个学习源码的切入点

通过寻找程序启动的入口，对入口断点调试，可以从源头了解框架的启动流程和运行原理。

可以通过打断点，然后通过调用栈逆向寻找入口；可以找网上的博客的源码分析找到入口打断点。

<video class="video-0" data-index="0" webkit-playsinline="" playsinline="" x5-playsinline="" x5-video-player-type="h5" x5-video-player-fullscreen="true" poster="img/5cb037580001039a06700377.jpg" controls="controls" controlslist="nodownload" src="img/H.mp4" style="">  Your browser does not support the video tag. </video>
##### 4.3 阅读源码时要重视函数的命名

往往优秀的源码函数命名都非常贴切。可以通过 IDEA 的 structure，来了解源码中某个类的核心函数。

核心类有哪些核心的函数，这些函数的功能又是什么，对学习源码帮助很大。

通过单个函数快速了解其意图，对学习源码帮助很大。



##### 4.4 多看函数列表

在看源码时建议打开函数列表，进入某个类时优先看该类有哪些公有函数。

![图片描述](img/5dedbb8d0001923c09830569.png)
这样做有助于帮助你从整体了解该类，更全面地了解一个类的功能。



##### 4.5 阅读源码时要重视源码的注释

优秀的开源项目的类、函数甚至成员变量都会有非常详尽的注释。

注释可以快速帮助我们理解源码，帮助我们了解一些重要细节。

比如很多代码会给出其核心步骤，此时一定要先阅读函数上面的注释和内部给出的核心步骤再去读源码。

比较典型的一个案例 `java.util.concurrent.ThreadPoolExecutor#execute` ：

```java
/**
 * Executes the given task sometime in the future.  The task
 * may execute in a new thread or in an existing pooled thread.
 *
 * If the task cannot be submitted for execution, either because this
 * executor has been shutdown or because its capacity has been reached,
 * the task is handled by the current {@code RejectedExecutionHandler}.
 *
 * @param command the task to execute
 * @throws RejectedExecutionException at discretion of
 *         {@code RejectedExecutionHandler}, if the task
 *         cannot be accepted for execution
 * @throws NullPointerException if {@code command} is null
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```



##### 4.6 关注目标类继承的类或者实现的接口

目标类的父类和实现的接口是研究该类功能和特征的重要突破口。

借助前面章节讲到的调试技巧，查看调用栈，运行表达式等可以极大地帮助我们理解源码。通过 IDEA 提供的类图功能，可以帮助我们理解不同类之间的关系。

如下图所示，通过 IDEA 自带的类图工具绘制出 fastjson 核心类之的类图，通过类图的选项来控制显示的内容和可见性。

![图片描述](img/5dedbb6300013eff12390664.png)



##### 4.7 其它

大家可以使用前面调试章节所学到的查看调用栈、设置条件断点、查看加载的对象等调试功能来帮助大家学习源码。

大家还可以跟着某个框架的专栏作者的思路去深入学习某个具体框架的源码。



#### 5. 总结

本节主要讲述如何阅读源码，讲到了阅读源码的思路和一些技巧。希望通过本文的介绍大家可以更高效地阅读源码，提高进阶的速度。



### 代码重构的正确姿势



#### 1. 前言

在软件迭代过程中常常会因为原来的功能有 BUG、无法满足新的需求、性能遇到瓶颈等原因为需要对代码进行重构。

那么：

- 为什么要重构？
- 如何保证重构代码的正确性？
- 有哪些重构技巧？

这三个关键问题都是本节的重点探讨的内容。



#### 2. 什么是重构？何时重构？



##### 2.1 什么是重构？

想了解为何要重构以及如何重构，就要先搞清楚什么是重构。

> 重构（ refactoring）是这样的一个过程：在不改变代码外在行为的前提下，对代码进行修改，以改进程序的内部结构。本质上讲，重构就是在代码写好之后，改进它的设计。 – 《重构》[1](https://www.imooc.com/read/55/article/1163#fn1)



##### 2.2 何时动手重构？



###### 2.2.1 添加新功能的时候

当因为新的需求要为系统添加新功能时，可能会发现很多问题。

比如发现不同的类需要使用同一段代码，而这段代码在之前的一个类中；发现分支条件越来越多，难以维护；发现随着功能的增强，函数的参数列表越来越长，代码长度太长难以理解等。

可以借助开发新功能的时机去对代码进行重构。



###### 2.2.2 修复错误时重构

当我们收到一份来自测试或者技术支持提过来的 “编码缺陷” 的 jira 时几乎就意味着我们要重构代码了。

可能是接口的结果不符合预期，也可能是接口的性能达不到要求。



###### 2.2.3 代码审查时重构

很多公司都会有代码审查机制，复杂、重要的项目都要通过代码审查（ Code Review） 后才能上线。

在代码审查阶段，代码审查人员可能对我们代码的可读性、可维护性、代码的性能等进行评价并给出建议。

如果代码审车人员给出了比较合理的建议，此时就要对有问题的代码进行重构。



#### 3. 如何保证重构代码的正确性？

**重构技巧千万种，保证正确性是关键。** 那么如何保证重构代码的正确性呢？

正如单元测试的章节所讲的一样，**单元测试是保证代码正确性的强有力保证**。

《重构》第二版 [2](https://www.imooc.com/read/55/article/1163#fn2)“重构第一步” 小节有这样一段描述：

> 进行重构时，我们需要依赖测试。我们将测试视为 bug 检查器，它们能够保护我们不被自己犯的错所困扰。
>
> 把我们想表达的目标写两遍 -- 代码里写一遍，测试里再写一遍 -- 我们的错误才能骗过检测器。这降低了我们犯错的概率。尽管编写测试需要花费时间，但却为我们节省下可观的调试时间。

因此要保证重构的代码都可以通过测试，如果前人并没有编写对应的单元测试，可以在重构时补上对应的单元测试。



#### 4. 一些重构技巧

Java 代码的重构主要包括以下几个方面：代码的 “坏味道”，对象之间的重构，数据的重构，函数调用的重构和表达式简化的重构。



##### 4.1 代码的坏味道

代码的坏味道有很多种，常见的包括：重复代码，过长的函数，过大的类，过长的参数列表，过多的注释等。

**重复代码**通常有 3 种情况，

1、**同一个类的多个函数包含重复代码**， 此时可以将公共代码提取为该类的私有函数，在上述函数中调用；

2、**互为兄弟的子类之间包含相同的代码**，此时应该将重复代码上移到父类中；

3、**两个毫不相关的出现重复代码**，此时应该将公共代码抽取到一个新类中。

比如在实际开发中，经常需要根据将某个字段和枚举的值进行比较，可能频繁出现如下代码：

```java
Integer someType = xxxDTO.getType();
// 第一种形式
if (CoinEnum.PENNY.getValue() == someType) {
    // 代码省略
}

// 第二种形式
if (CoinEnum.PENNY == CoinEnum.getByValue(someType)) {
    // 代码省略
}
```

那么如何变得更优雅呢？

项目中多处需要执行批量逻辑，可能需要对接口数量做限制，会在项目中多处出现这种代码：

```java
 public <T> void aRun(List<T> dataList) {
   
   if (CollectionUtils.isEmpty(dataList)) {
          return;
   }
   int size = 10;
 
   // 每 10 个元素为一组执行一次
    Lists.partition(dataList, size).forEach((data) -> someRun(dataList));
 }

 private <T> void someRun(List<T> dataList) {
       // 省略
 }
```

此时可以将其封装到工具类中：

```java
public static <T> void partitionRun(List<T> dataList, int size, Consumer<List<T>> consumer) {
    if (CollectionUtils.isEmpty(dataList)) {
        return;
    }
    Preconditions.checkArgument(size > 0, "size must > 0");
    Lists.partition(dataList, size).forEach(consumer);
}
```

如果在项目中多个地方需要类似的逻辑， 则直接调用该工具类即可：

```java
// 每批 10 个
ExecuteUtil.partitionRun(mockDataList, 10, (eachList) -> a.someRun(eachList));

// 每批 20 个
ExecuteUtil.partitionRun(mockDataList, 20, (eachList) -> b.otherRun(eachList));
```

如果**函数过长**，读懂函数的逻辑将变得非常困难，接手代码的人需要花费较多时间才能读懂这些代码。

在工作中，如果接手的代码某一行报错，但是代码行数很多，一般需要读懂整个函数逻辑才敢动手修改，是一件非常痛苦的事情。根据《手册》 "【推荐】单个函数总行数不超过 80 行” [3](https://www.imooc.com/read/55/article/1163#fn3) 的建议，需要将大函数拆分成多个子步骤（函数）。最好的办法是搞清楚该函数分为几个步骤，分别将每个子步骤提取为一个子函数即可。

如果**类过大**，通常是函数太多，成员变量过多。如果是函数太多，通常可以根据将函数归类，拆分到不同的类中，一个常见的做法是将 `OrderService`` 拆成 OrderSearchService` 和 `OrderOperateService` 分别承担订单的搜索和非搜索业务。如果是成员变量过多，则需要考虑是否应该多个成员变量抽取到某个类中，后者一部分成员变量是否应该属于某个类，通过将新类当做成员变量来消减成员变量的数量。

如果**函数的参数较长**，传参时需仔细核实参数列表以避免误传。如果对外暴露的接口，需要新增一个属性时，为了避免修改签名让二方被迫跟着修改调用的代码，就需要新增一个接口，这种不优雅的方案。根据《手册》的分层领域模型规约部分的建议，**应该将请求的参数封装成查询对象**。这也是一个宝贵的开发经验，尤其是暴露给二方 RPC 接口时，如果未来可能修改参数，尽量使用对象来接收参数，避免因函数签名不同而导致错误。

如果**代码中的注释过多**，应该简化注释，尽量只在关键步骤，特殊逻辑上添加注释，应该使用变量和函数名来表意。



##### 4.2 重新组织函数

**当函数中条件表达式较为复杂时**，应该将复杂表达式或者其中一部分放到临时变量中，并通过变量名来表达其用途，也可以将部分表达式在一起组成一个含义，还可以将其封装到函数中。

可以参见 spring `TypeConverterDelegate#convertToTypedCollection` 源码：

```java
// 提取为变量
boolean approximable = CollectionFactory.isApproximableMapType(requiredType);
if (!approximable && !canCreateCopy(requiredType)) {
   if (logger.isDebugEnabled()) {
      logger.debug("Custom Map type [" + original.getClass().getName() +
            "] does not allow for creating a copy - injecting original Map as-is");
   }
   return original;
}
// 提取为函数
	private boolean canCreateCopy(Class<?> requiredType) {
		return (!requiredType.isInterface() && !Modifier.isAbstract(requiredType.getModifiers()) &&
				Modifier.isPublic(requiredType.getModifiers()) && ClassUtils.hasConstructor(requiredType));
	}
```

**如果不在循环中对一个含义不明确的临时变量多次赋值时**，需对每一次都创建独立的临时变量。

如下列代码使用一个临时变量表达多种含义：

```java
int temp = array.length;
// 省略中间代码
temp = user.getAge();
```

应该修改为：

```java
int length = array.length;
// 省略中间代码
int age = user.getAge();
```

如果**调用二方批量接口响应很慢容易超时**，除了可以像 4.1 重复代码所给出的示例一样，将其**改为小批次调用，并将小批次调用的结果进行聚合**。

通过封装成工具函数实现复用，可以通过控制 size 来避免接口超时：

```java
public static <T, V> List<V> partitionCall2List(List<T> dataList, int size, Function<List<T>, List<V>> function) {

    if (CollectionUtils.isEmpty(dataList)) {
        return new ArrayList<>(0);
    }
    Preconditions.checkArgument(size > 0, "size must > 0");

    return Lists.partition(dataList, size)
            .stream()
            .map(function)
            .filter(Objects::nonNull)
            .reduce(new ArrayList<>(),
                    (resultList1, resultList2) -> {
                        resultList1.addAll(resultList2);
                        return resultList1;
                    });


}
```

为了获取更快的响应速度，可以使用并发或并行特性:

```java
public static <T, V> List<V> partitionCall2ListWithCompletable(List<T> dataList,
                                                     int size,
                                                     ExecutorService executorService,
                                                     Function<List<T>, List<V>> function) {

    if (CollectionUtils.isEmpty(dataList)) {
        return new ArrayList<>(0);
    }
    Preconditions.checkArgument(size > 0, "size must >0");

  // 异步调用并获取CompletableFuture对象列表
    List<CompletableFuture<List<V>>> completableFutures = Lists.partition(dataList, size)
            .stream()
            .map(eachList -> {
                if (executorService == null) {
                    return CompletableFuture.supplyAsync(() -> function.apply(eachList));
                } else {
                    return CompletableFuture.supplyAsync(() -> function.apply(eachList), executorService);
                }

            })
            .collect(Collectors.toList());

// 等待全部完成
    CompletableFuture<Void> allFinished = CompletableFuture.allOf(completableFutures.toArray(new CompletableFuture[0]));
    try {
        allFinished.get();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
  
  // 组合结果
    return completableFutures.stream()
            .map(CompletableFuture::join)
            .filter(CollectionUtils::isNotEmpty)
            .reduce(new ArrayList<V>(), ((list1, list2) -> {
                list1.addAll(list2);
                return list1;
            }));
}
```

还有无数种可以重构的情况，更多重构的场景和范例请参考《重构》这本经典著作，在编码过程中认真体会和运用。

**在平时开发时，在满足功能需求的基础上要注重代码的性能**。

比如下面这段代码：

```java
public List<String> getImages(String type) {

    List<String> result = new ArrayList<>();

    if ("10*20".equals(type)) {

        result.add("http://xxxxxximg1.png");
        result.add("http://xxxxxximgx.png");
        // 省略其他
    } else if ("10*30".equals(type)) {
        result.add("http://yyyyimg1.png");
        result.add("http://yyyyimgy.png");
        // 省略其他
    }
    return result;
}
```

我们可以看到图片的内容是固定的，不需要每次都要查询，上面的写法每个请求都要创建一个 List 将对应的图片塞进去再返回，完全没有必要。

可以参考下面的代码进行重构：

```java
private static Map<String, List<String>> images;

static {
    images = new HashMap<>();

   // 根据元素的个数设置初始长度
    List<String> first = new ArrayList<>(16);
    first.add("http://xxxxxximg1.png");
    first.add("http://xxxxxximg2.png");
    // 省略其他
    List<String> second = new ArrayList<>();
    second.add("http://yyyyimg1.png");
    second.add("http://yyyyimg2.png");
    // 省略其他

    images.put("10*20", first);
    images.put("10*30", second);
    images = Collections.unmodifiableMap(images);

}

public List<String> getImages(String type) {
    if (StringUtils.isBlank(type)) {
        return new ArrayList<>();
    }
    return images.getOrDefault(type, new ArrayList<>());
}
```

这样每次请求都会从 “缓存” 中获取，而且为了防止 map 被修改，将其设置为 unmodifiableMap，而且使用 Map 接口的 getOrDefault 功能，大大简化了代码。



##### 4.3 线程安全问题

当多线程共享变量是，要特别注意线程安全问题。

请看下面的例子：

```java
@Service
public class DemoServiceImpl implements DemoService{

    private static List<String> data;

    private List<String> doGetData() {
      // 第一处代码
        if (data == null) {
            data = new ArrayList<>();
            data.add("a");
            data.add("b");
            data.add("c");
        }
        return data;
    }

    @Override
    public List<String> getData(String param) {
      // 省略其他
        List<String> data = doGetData();
      // 省略其他
    }
}
```

假设有多个线程并发调用 doGetData 函数，最初的前两个线程极短时间内依次走到第一处代码的判断处，都会进入 if 代码块。

如果第一个线程在调用 data.addd (“c”) 后，如果第二个线程执行 data = new ArrayList<>(); 第一个线程返回 data 时，该集合内没有元素（元素丢失）。

针对该示例代码的情况，我们可以通过静态代码块来构造数据，也可以通过双重检查锁来实现。

参考修改 1：

```java
private static List<String> data;

static {
    data = new ArrayList<>();
    data.add("a");
    data.add("b");
    data.add("c");
    data = Collections.unmodifiableList(data);
}
```

参考修改 2：

```java
  private static final Object LOCK = new Object();
 // 第 1 处代码
  private static volatile List<String> data;

  private List<String> doGetData() {
       if (data == null) {
            synchronized (LOCK) {
                if (data == null) {
                  // 第 2 处代码
                    ArrayList<String> inner = new ArrayList<>();
                    inner.add("a");
                    inner.add("b");
                    inner.add("c");
                    // 第 3 处代码
                    data = Collections.unmodifiableList(inner);
                }
            }
        }
        return data;
   }
```

我们在第 1 处代码加上 volatile 关键字来保证可见性，我们在第 2 处代码创建一个局部变量，避免使用 data = new ArrayList<>(); ，因为这样会导致还没添加数据就已经创建了对象，另外一个线程并发访问时直接进行最外层判断时就满足 data != null 返回没有元素的 data 集合。



##### 4.4 使用权威的工具类

我们尽量使用 JDK 封装好的类，使用大公司开源的工具类，避免重复劳动。

平时可以多去 commons-lang3 、commons-collections4 、 guava 等知名工具类框架中了解其提供的简单实用的工具类。

比如让当前线程 sleep 一段时间，不要用数字自行计算：

```java
Thread.sleep(3 * 1000*60);
```

而应该使用时间相关的类：

```java
TimeUnit.MINUTES.sleep(1);
```

如开发中计算耗时，通常获取开始和结束的时间，然后结束时间减去开始时间：

```java
@Test
public void useTimeStamp() {
    long start = System.currentTimeMillis();
    // 省略一些代码
    long end = System.currentTimeMillis();
    System.out.println(end - start);
}
```

应该使用 StopWatch 类，不仅简单方便，而且该类还提供了更多强大功能：

```java
@Test
public void useStopWatch() {
    StopWatch stopWatch = StopWatch.createStarted();
    // 省略一些代码
    System.out.println(stopWatch.getTime());
}
```

非常建议大家在开发中使用第三方工具类时能够主动进入其源码，打开函数列表，去查看里面提供的核心工具类，有时候会有意外发现。



#### 5. 总结

本文主要讲述什么是重构，何时重构，并选取几个典型场景为例说明如何重构。更多重构的场景和范例请参考《重构》这本经典著作。另外推荐《编写可读代码的艺术》、《代码简洁之道》这两本书，它们都是提高代码可读性和重构代码的不错参考资料。



### Code Review的正确姿势



#### 1. 前言

相信很多从事 Java 开发行业的同学都听说过代码审查 (Code Review)。那么是否思考过下面几个问题？

- 代码审查是什么？为啥要代码审查？
- 代码审查审查什么？谁来审查？
- 如何进行代码审查呢？



#### 2.What? Why?Who?



##### 2.1 什么是代码审查？

> 代码审查也叫代码复查，即通过阅读代码的方式来检查代码是否符合要求。

通俗来讲，我们代码开发完成以后，项目发布之前，需要让其他人阅读我们的代码，看是否符合要求，是否存在隐患等。

代码审查的目标主要有：

- 保证每一次上线时，至少会有两个人能够理解该代码的逻辑。
- 保证每次代码审查后的结果都会落到实处，有跟进有解决。



##### 2.2 为什么要代码审查？

代码审查是提高代码质量，提前发现 BUG ，统一团队代码规范的一个重要途径。

俗话说：“不识庐山真面目，只缘身在此山中”。

由于第一人称视角和个人的习惯难改，经验有限的缘故，我们通常很难发现自己编码中存在的问题。

因此团队中开发经验丰富的其他成员对代码进行把关，更容易发现代码中存在的功能和性能问题，能够提前发现一些隐患。

通过代码审查可以检查团队成员是否按照团队的编码规范进行开发。

同时代码审查也是团队成员相互学习的一种重要方式，作为代码审查人员可以学习团队其他成员的编程技巧，设计方案等，是学习进阶的一个极佳方式。



##### 2.3 谁来审查？

每个公司和团队的规定可能有所不同。

常见的代码审查人员为团队的技术主管，团队中有丰富开发经验的老员工。

也有些团队对于日常的小需求会让开发人员自己找其他熟悉相关背景的同事进行代码审查。

这里分享一个**非常重要的经验**，我们在开发过程中就可以将自己的开发分支和 master 分支的最新代码进行比对，按照代码审查的内容对自己代码进行审查，这样能够及时发现问题，尽可能避免将问题遗留在 “正式的代码审查阶段”。



##### 2.4 代码审查，审查什么？



###### 2.4.1 审查设计规范

- 代码是否符合面向对象的五大原则原则？是否符合领域设计的原则？

> **面向对象的五大原则**：单一职责原则，开闭原则，里氏替换原则，接口隔离原则，依赖倒置原则。
>
> 如一个函数干了两件不相干的事情，应该拆分为两个函数。
>
> 如嵌套超过 3 层可以使用卫语句、策略模式、状态模式等。

- 是否使用了某种设计模式？该设计模式使用是否恰当？
- 新的代码是否符合团队的代码规范？
- 代码位置是否正确？如用户相关的代码是否放到了用户服务中？
- 新代码是否重用了已有代码？新代码是否提供了可重用的代码？结合前面章节学到的重构原则，是否需要重构？
- 新代码是否有过度设计的现象？是否违背 YAGN 原则？

> **YAGN** 即 “You aren’t gonna need it”。只有在需要的时候程序员才应该编写相应的代码。[1](https://www.imooc.com/read/55/article/1164#fn1)



###### 2.4.2 审查可读性和可维护性

- 属性、变量、参数、方法类的命名是否能够反映出其要表达的真实含义？
- 是否能够理解自己阅读的代码？
- 是否能够理解测试代码？
- 测试是否覆盖了所有分支？测试是否覆盖了正常和异常情况？是否有一些未考虑的情况？
- 异常信息是否容易理解？
- 代码片段、文档、注释是否有令人困惑的部分？



###### 2.4.3 审查是否有错误

- **代码是否符合需求？** 测试代码编写的是否正确？
- **代码中是否有潜在的 BUG？** 是否校验错了参数？判断条件是否有误？
- 代码是否有安全问题？如 SQL 注入。
- 代码是否存在性能问题？是否有不必要的数据库调用、IO 读写操作和远程调用？
- 编写代码的开发人员是否更新了相关文档？
- 是否有明显的错误？如是否会偶发性指向测试库？是否需要调用真实服务时，却没有调用真实的服务，而采用了硬编码方式 mock 了接口？



###### 2.4.4 审查测试代码

- 源码作者是否为新的代码或修复的代码编写了测试代码？

- 测试是否覆盖到了容易出错的部分或者复杂的部分？

- 测试是否验证了代码的正确性？

- 是否能够想出没有被当前测试代码覆盖的场景？

- 是否测试了代码的限制？如批量查询接口只支持 100 个元素，是否被测试到了？

- 是否涵盖了安全方面的测试？

- 是否有性能测试？

  

  ###### 2.4.5 审查性能方面

  - 新的代码中某些函数是否有性能要求？是否验证了该函数的性能？
  - 新的代码是否降低了之前的性能？
  - 是否有不必要的外部服务调用？如不必要的调用数据库，不必要的外部服务调用。
  - 新代码是否有可能引起内存泄露的情况？
  - 是否有可能引起内存占用无限增长的情况？
  - 代码中使用的资源是否正确关闭？如创建的连接或者流。
  - 资源池的配置是否正确？
  - 是否使用了在多线程场景下可能引发性能问题的类？

> 正如《手册》并发处理章节 第 15 条 Random 如果被多线程共享，虽然线程安全，但是因为竞争同一个 seed 会导致性能下降 [2](https://www.imooc.com/read/55/article/1164#fn2)。

- 主要可能引发性能问题的常见场景，如果用到了反射，要思考是否必须要用反射？要思考提供的接口如果超时会怎样？超时时间设置为多少更合理？是否使用了并行特性？是否在没必要的地方使用了并行特性？



###### 2.4.6 审查正确性

- 多线程环境下数据结构是否选择正确？

> 如在多线程环境下共享变量使用非线程安全的类，就可能出现线程安全问题。
>
> 如可以使用 `Collections.sychronizedList()` 来创建线程安全的 `List`. 如果读的频率远大于写的频率，可以使用 `CopyOnWriteArrayList`.
>
> 再如多线程条件下， `SimpleDateFormat` 存在线程安全问题，为了避免线程安全问题可以使用 `ThreadLocal` 的方式。

- 代码中锁的使用是否正确？
- 代码使用的数据结构是否正确？如需要随机访问，却使用了 `LinkedList` , 需要保证唯一性却使用了 List.
- 日志的级别设置是否正确？



###### 2.4.7 审查安全性

- 是否缺乏安全校验？如外部服务调用的结果没有判空直接使用导致空指针错误。
- 批量查询接口是否有对查询数量进行限制？
- 敏感操作是否做了权限校验？
- 在使用平台资源时，是否做了限制？如短信、邮件是否做了数量限制、疲劳控制？下单、支付是否做了幂等校验？
- 是否存在 SQL 注入的可能性？
- 日志是否脱敏？
- 发帖、评论、发送即时消息等用户生成内容的场景是否实现了防刷，风控？
- 开发人员是否开发了一些危险后门？

更多具体案例大家可以参考 [Upsource - Docs & Demos](https://www.jetbrains.com/upsource/documentation/)



##### 2.5 如何审查



###### 2.5.1 代码审查工具

最简单直接的代码审查工具就是 GIT ，git 提供了强大的 diff 功能，通常代码审查人员会将所要审查的分支和 master 分支进行对比。

也可以使用类似 Upsource （JetBrains 出品的一款知名的代码审查和项目分析工具）。

建议在代码审查前自己预先使用 findbugs 等静态检查工具，检查出一些潜在的风险提前修改。



###### 2.5.2 代码审查的形式

代码审查的形式每个公司和团队也千差万别，这里介绍常见的利用简单的代码审查形式。

**代码审查的形式**

有些团队的代码审查人员通过需求文档了解项目的需求，通过技术文档了解对方的设计，然后拉取对方的开发分支和 master 进行比较，按照上述的审查内容进行审查，对发现的问题进行记录，并约时间双方进行沟通。

也有些团队可能将审查人员和开发人员约在一起，开发人员负责介绍项目的需求、技术文档，然后介绍自己代码修改点，自己代码的核心实现。由代码审查人员随时提问，开发人员现场解答，并对待修改的地方做记录，根据代码审查人员的建议做出修改。代码审查后要修复审查中遇到的问题，尽量当天完成。下一次代码审查时，需先检查之前的问题修复情况。再次记录此次代码审查的问题。

**具体原则**

小项目代码审查：只要有新写的或者修改的代码都可以拿出来审查；代码审查至少两人在场，场地不限，时长尽量短；聚焦核心代码、核心业务逻辑，关键点。

大项目的代码审查：尽量找个会议室，更多人参与其中，和 master 全量 diff。



#### 3. 总结

本节主要讲述了代码审查的概念，重点解答了：为什么要代码审查？代码审查审查什么内容？如何进行代码审查。

希望大家不要坐等别人对自己代码进行审查，可以养成在开发过程中对自己代码进行自我审查的习惯，多将自己的开发分支代码和最新的 master 代码进行比对，避免将太多问题暴露到软件生命周期的靠后的阶段，降低修改的成本。



### 授人以渔



#### 1、前言

最近有一个朋友询问一个《Effective Java》[1](https://www.imooc.com/read/55/article/1347#fn1) 中的 “诡异问题”，表示看不懂里面的讲解。

本节将以该问题为素材，灵活运用本专栏所介绍的各种方法来研究这个问题。



#### 2、问题描述

《Effective Java》 第二版，第 16 条： 复合优于继承小节提到：

> 为了调优该程序的性能，需要查询 `HashSet` ，看看自从被创建以来添加了多少个元素。为了提供这种功能，我们得用一个 `addCount` 变量记录插入的元素数量，并提供一个获取该变量数值的方法。
>
> HashSet 类包含两个可以增加元素的方法： `add` 和 `addAll`，因此两个方法都要覆盖。

文中给出了下面的示例代码：

```java
import java.util.Collection;
import java.util.HashSet;

public class InstrumentedHashSet<E> extends HashSet<E> {

    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

这段代码看似没啥问题，添加一个元素时 `addCount` 加 1 ，添加多个元素时 `addAll` 函数中会加上参数集合的长度。

接下来我们编写单元测试代码，来验证功能是否正确：

```java
public class InstrumentedHashSetTest {

    @Test
    public void testAddCount() {
        List<String> stringList = Arrays.asList("Snap", "Crackle", "Pop");
        InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
        s.addAll(stringList);
        Assert.assertEquals(stringList.size(), s.getAddCount());
    }
}
```

我们运行该程序，发现断言失败，显示期待的值是 3 ，而实际值却是 6 :

> java.lang.AssertionError:
> Expected :3
> Actual :6

此时多人会奇怪，为啥结果会是 6 呢？

希望大家在读下一部分之前先停下来想想为什么会这样，然后再去和后面的解读核对，这样印象会更深刻一些。

否则就像看着答案做题一样，看的时候觉得啥都会，其实没有真正掌握。



#### 3. 解读



##### 3.1 猜想验证

遇到问题时尽量要：先猜想，后验证。

遇到问题根据已有知识去 “猜想”，然后去验证和实际答案对比，才能发现自己理解不到位的地方，可以纠正自己理解。

这样学习才更有效，印象也会更加深刻。

> 很多人很羡慕别人能够快速准确地定位问题，但是平时自己却不能够通过简单的问题锻炼自己的猜想验证能力，直接看结论很难发现自己知识的欠缺，关键时自然无法快速准确地分析问题。

根据这种表现我们作出两个推测：

1、`addAll` 函数被调用了 2 次；

2、`addAll` 函数的执行，最终又调用了 3 次 `add` 函数。

根据源码可知：第一种猜测，显然不成立。

使用排除法，结论就显而易见了，只有第二种可能性。

**可是为什么 `add` 函数会被调用 3 次呢**？

考试可以用排除法，但是研究不可取，我们还需要通过各种方法去验证。

> 很多人学习不够深入的原因之一就是看到某个结论就 “记住”，从来不去验证。
>
> 如果看到的博客、图书讲解有误，自己将会被带偏；如果讲解层次比较浅显，自己也将停留在这个层次。



##### 3.2 类图大法

我们可以先使用 IDEA 自带的类图工具，来查看我们自定类的继承关系。
![图片描述](img/5df987680001c75817783280.png)

通过上述类图，我们可知：我们自定义的类，通过继承 `HashSet` 实现了 `Cloneable` 接口，即支持克隆；通过继承 `HashSet` 实现了 `Serializable` 即支持 Java 序列化，顶层实现了 `Iterable` 接口，即支持迭代器方式遍历元素。

当我们对一个类不足够熟悉时使用类图，我们可以快速了解它。可以通过观察函数名、参数列表和返回值等，来快速找到我们需要的函数。

我们可以点击源码中的 `super.addAll` 查看调用的函数源码，发现来自 `java.util.AbstractCollection#addAll`。

我们还可以通过函数列表来了解单个类，了解除了常用功能之外还有哪些函数，它们的作用是什么。

![图片描述](img/5df987840001852112330817.png)
在此，建议大家在平时学习和开发时，如果任务并不是特别繁重，可以偶尔进入常用的 JDK 源码和第三方框架类中，查看它们的函数列表，了解该类提供的函数，容易发现一些好用的但是自己不常用的函数，能够了解其底层实现。这样积少成多，会学的越来越深入。

> 举一个常见的现象：
>
> 1、很多人经常使用 guava、commons-lang、commons-collections4 等知名三方工具类，但是永远都是用最常见的那几个工具函数。由于没有养成去源码中看函数列表的习惯，实际开发时会发现自己会重复造轮子，而自己造的轮子不管从健壮性还是代码的优雅程度都不如三方工具类，而且还浪费了不少时间；
>
> 2、很多人没有随手养成看源码的习惯，总是依赖看博客、看书来学习，导致缺少了一个非常好的学习途径。

如要移除本例中的字符串长度大于 3 的元素，很多人可能会 “颇有自信” 地用迭代器这么写：

```java
List<String> stringList = Arrays.asList("Snap", "Crackle", "Pop");
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(stringList);

Iterator<String> iterator = s.iterator();
while (iterator.hasNext()){
    String next = iterator.next();
    if(StringUtils.isNotBlank(next) && next.length() >3){
        iterator.remove();
    }
}
System.out.println(s);
```

但是通过函数列表我们发现有一个 `removeIf` 函数，我们进入函数看源码：

```java
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean removed = false;
    final Iterator<E> each = iterator();
    while (each.hasNext()) {
        if (filter.test(each.next())) {
            each.remove();
            removed = true;
        }
    }
    return removed;
}
```

我们 “惊讶地” 发现，和我们要写的代码 “惊人地” 一致，既然都给我们封装好了，我们可以通过它来简化代码：

```java
List<String> stringList = Arrays.asList("Snap", "Crackle", "Pop");
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
s.addAll(stringList);

s.removeIf(next -> StringUtils.isNotBlank(next) && next.length() > 3);
System.out.println(s);
```

通过这个简单的案例，希望大家可以养成好的学习习惯，这将有助你快速进阶。

接下来我们回归正题，继续研究我们前面提到的问题。



##### 3.3 源码大法

其实看官方解释之前，最好自己动手根据我们已经掌握的方法先去研究这个问题，然后再去验证。

既然出现问题，我们就要分析问题。

既然调用了 `supper.addAll()` ，该函数继承自 java.util.AbstractCollection，我们可以先看下它的实现：

```java
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```

我们发现，这里会通过 foreach 循环的方式分别调用 add 函数来添加每个元素。

然后我们进入默认的 add 函数:

```java
public boolean add(E e) {
    throw new UnsupportedOperationException();
}
```

发现该函数默认会抛出不支持的操作异常（UnsupportedOperationException ）。

我们还发现父类 `HashSet` 重写了 add 函数 （java.util.HashSet#add）：

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

我们自定义的类也重写了 `add` 函数（com.imooc.basic.inheritance.InstrumentedHashSet#add）：

```java
@Override
public boolean add(E e) {
    addCount++;
    return super.add(e);
}
```

根据运行结果，我们推测调用到了 `InstrumentedHashSet` 类重写的 `add` 函数。

看到这里**我们得到了一个启示**：

我们在学习和研究问题时，要养成主动去看源码的习惯。

很多人想学得扎实一些，看各种书，但是从来不会主动去看源码，导致记住的东西容易忘记，很多知识知其然而不知其所以然。

看到这里**该同学又产生了一个疑问**：

为啥调用 `supper.addAll` 里面自定义类的 `add` 函数，没调用父类的 `add` 函数呢？

其实从这个疑问中可以看出该同学基础并不扎实，对面向对象的多态特性理解不够透彻。



##### 3.4 断点调试大法

有问题没关系， 为了解开困惑，我们继续用我们已经掌握的方法来研究。

接下来我们用断点调试大法来分析这个问题。

我们先思考一个问题：

```
java.util.AbstractCollection#addAll` 并不是静态函数，属于实例函数，也就是说这里的 `add(e)` 等价于 `this.add(e)
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    for (E e : c)
        if (add(e))
            modified = true;
    return modified;
}
```

那么这里的 this 是什么（函数所属的对象是谁）？
![图片描述](img/5df9879c0001465a12080501.png)

单步或者通过条件断点走到这里时我们发现，此时 this 的类型为 `InstrumentedHashSet` 。

因此，这里的 this 应该是调用测试代码中通过 new 关键字构造的对象：

```java
InstrumentedHashSet<String> s = new InstrumentedHashSet<>();
```

调用的就应该是 `com.imooc.basic.inheritance.InstrumentedHashSet#add` 函数。

继续单步跟到函数内部，发现的确如此。

自此，通过代码调试我们找到了 addCount 值为 6 的原因。

**那么是不是这个问题到这里就结束了呢**？

当然不是…

**问题来了：**

如果我们不借助调试，如何确定它最终调用的是 `com.imooc.basic.inheritance.InstrumentedHashSet#add` 函数，而不是直接调用父类（HashSet）中的 `add` 函数呢？

很多人可能会回答：“显而易见”、“博客上这么说的”、“看书上就这么讲的”、“老师就是这么教的”，囧…

当你回答上述答案，而不再深究时，就说明你将在很长一段时间停留在这个层次。

这种 “显而易见”，这种 “书上说的” 将是阻碍我们进一步深入研究的一个很大的障碍，很多人都会存在这种问题，希望大家警惕。

很多 “显而易见” 的知识，并没那么 “理所当然”，请看下面的分析。



##### 3.5 反汇编和 JVMS 大法

我们继续从字节码层面研究这个问题。

我们先对代码进行编译： `javac InstrumentedHashSet.java`

然后对编译后的字节码文件进行反汇编： `javap -c InstrumentedHashSet

得到如下的反汇编代码：

```java
public class com.imooc.basic.inheritance.InstrumentedHashSet<E> extends java.util.HashSet<E> {
  public com.imooc.basic.inheritance.InstrumentedHashSet();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/util/HashSet."<init>":()V
       4: aload_0
       5: iconst_0
       6: putfield      #2                  // Field addCount:I
       9: return

  public com.imooc.basic.inheritance.InstrumentedHashSet(int, float);
    Code:
       0: aload_0
       1: iload_1
       2: fload_2
       3: invokespecial #3                  // Method java/util/HashSet."<init>":(IF)V
       6: aload_0
       7: iconst_0
       8: putfield      #2                  // Field addCount:I
      11: return

  public boolean add(E);
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // Field addCount:I
       5: iconst_1
       6: iadd
       7: putfield      #2                  // Field addCount:I
      10: aload_0
      11: aload_1
      12: invokespecial #4                  // Method java/util/HashSet.add:(Ljava/lang/Object;)Z
      15: ireturn

  public boolean addAll(java.util.Collection<? extends E>);
    Code:
       0: aload_0
       1: dup
       2: getfield      #2                  // Field addCount:I
       5: aload_1
       6: invokeinterface #5,  1            // InterfaceMethod java/util/Collection.size:()I
      11: iadd
      12: putfield      #2                  // Field addCount:I
      15: aload_0
      16: aload_1
      17: invokespecial #6                  // Method java/util/HashSet.addAll:(Ljava/util/Collection;)Z
      20: ireturn

  public int getAddCount();
    Code:
       0: aload_0
       1: getfield      #2                  // Field addCount:I
       4: ireturn
}
```

从上述代码，我们看到，super 关键字在编译器已经确定要调用的函数。即 `super.add(e);` 调用的是 `java.util.HashSet#add` 函数，`super.addAll(c)` 调用的是 `HashSet.addAll` 实际上该函数是继承自 `AbstractCollection#addAll`。

既然 `super.addAll` 调到了 `AbstractCollection#addAll`，**那么这里的 `addAll` 中调用的 `add` 又是哪个函数呢**？

我们通过 `javap -c AbstractCollection` 反汇编 `AbstractCollection`：

```java
public abstract class java.util.AbstractCollection<E> implements java.util.Collection<E> {
  protected java.util.AbstractCollection();
    Code:
       0: aload_0
       1: invokespecial #2                  // Method java/lang/Object."<init>":()V
       4: return
 
  // 省略其他
  public boolean add(E);
    Code:
       0: new           #23                 // class java/lang/UnsupportedOperationException
       3: dup
       4: invokespecial #24                 // Method java/lang/UnsupportedOperationException."<init>":()V
       7: athrow

  public boolean addAll(java.util.Collection<? extends E>);
    Code:
       0: iconst_0
       1: istore_2
       2: aload_1
       3: invokeinterface #26,  1           // InterfaceMethod java/util/Collection.iterator:()Ljava/util/Iterator;
       8: astore_3
       9: aload_3
      10: invokeinterface #5,  1            // InterfaceMethod java/util/Iterator.hasNext:()Z
      15: ifeq          40
      18: aload_3
      19: invokeinterface #6,  1            // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
      24: astore        4
      26: aload_0
      27: aload         4
      29: invokevirtual #28                 // Method add:(Ljava/lang/Object;)Z
      32: ifeq          37
      35: iconst_1
      36: istore_2
      37: goto          9
      40: iload_2
      41: ireturn
  
}
```

理解该问题的关键就是要理解 invokevirtual 方法调用指令。

此时有些同学又要开始百度了！

NO，请打开我们 “随手必备的” [JVMS](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)[2](https://www.imooc.com/read/55/article/1347#fn2)，这一个习惯非常重要，直接决定你是掌握最准确的一手资料还是第 N 手资料。

在 JVMS 中搜索 invokevirtual 去了解即可找到它的含义和示例。

部分描述摘抄如下：

> The Java Virtual Machine gives special treatment to signature polymorphic methods in the *invokevirtual* instruction (§*invokevirtual*), in order to effect invocation of a *method handle*.
>
> JVM 可以通过 invokevirtual 实现多态函数逻辑。
>
> *invokevirtual* invokes an instance method of an object, dispatching on the (virtual) type of the object. This is the normal method dispatch in the Java programming language.
>
> *invokevirtual* 调用对象的实例函数，会根据对象的实际类型进行分派（即：在编译器不能确定最终调用的是子类还是父类的方法）。
>
> The difference between the *invokespecial* instruction and the *invokevirtual* instruction (§*invokevirtual*) is that *invokevirtual* invokes a method based on the class of the object. The *invokespecial* instruction is used to invoke instance initialization methods (§2.9) as well as private methods and methods of a superclass of the current class.
>
> `invokevirtual` 是基于对象的类来调用的方法的，而 `invokespecial` 用于调用实例初始化方法（构造函数），`private` 方法和当前类的父类的方法

另外，我们顺手把相关的 invokeinterface、invokespecial、invokestatic、invokedynamic 也初步了解一下：

- **invokestatic**：用于调用静态方法，即使用 static 关键字修饰的方法。这些方法在编译器就可以确定，运行期不会修改，因此方法调用指令中效率最高，属于静态绑定；
- **invokespecial**：用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所实现接口的默认方法。用在类加载时就能确定具体的方法，不需要等到运行时根据实际对象去调用该对象的函数；
- **invokevirtual**：用于调用非私有实例方法；
- **invokeinterface**：用于调用接口方法，在运行时确定一个实现此接口的对象；
- **invokedynamic**：用于调用动态方法，invokedynamic 把如何查找目标方法的决定权从虚拟机下放到了具体的用户代码中，为实现 lambda 表达式，实现动态语言等提供了便利。

> 注：
>
> 1、学习一个知识时，如果能主动学习相关知识，并对比他们的异同，可以学的更全面；
>
> 2、想深入学习更多虚拟机指令，可参考《深入理解 Java 虚拟机》[3](https://www.imooc.com/read/55/article/1347#fn3)、《Java 虚拟机规范》等资料。

有了这些知识背景，我们再回看我们反编译后的代码就容易理解了：

上述反汇编代码中的下面这行：

```java
12: invokespecial #4     // Method java/util/HashSet.add:(Ljava/lang/Object;)Z
```

表示我们自定义 `add` 函数中的 `super.add()` ，即使用 super 关键字调用父类实例方法： `HashSet#add` 函数

再回看前面提到的关键代码，`AbstractCollection#addAll` 调用的 `add` 函数的反汇编代码：

```java
 public boolean addAll(java.util.Collection<? extends E>);
    Code:
       0: iconst_0
       1: istore_2
       2: aload_1
       3: invokeinterface #26,  1           // InterfaceMethod java/util/Collection.iterator:()Ljava/util/Iterator;
       8: astore_3
       9: aload_3
      10: invokeinterface #5,  1            // InterfaceMethod java/util/Iterator.hasNext:()Z
      15: ifeq          40
      18: aload_3
      19: invokeinterface #6,  1            // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
      24: astore        4
      26: aload_0
      27: aload         4
      29: invokevirtual #28                 // Method add:(Ljava/lang/Object;)Z
      32: ifeq          37
      35: iconst_1
      36: istore_2
      37: goto          9
      40: iload_2
      41: ireturn
```

根据源码 26 行我们看到对象是 aload_0 加载到操作数栈的 this , 参数是 aload 4 加载的迭代器 next 返回的字符串对象（参见偏移量为 18-24 next 函数调用部分）。

> 注：aload_0 表示将第一个参数加载到操作数栈，非静态函数中第一个参数是 this。
>
> 有些同学可能对虚拟机指令不太熟悉，虚拟机指令最权威的参考资料还是《Java 虚拟机规范》，大家还可以参考周大大的《深入理解 Java 虚拟机》。

此时的 this 就是我们创建的 `InstrumentedHashSet` 类型的对象。

因为我们自定义类重写了 `add` 函数，根据 invokevirtual 的含义，我们就可以确认会调用到我们自定义的 `add` 函数。

至此，该问题就迎刃而解了。

**插曲：**

可能很多同学反汇编时会使用 -v 选项，来查看附加信息，

`javap -v AbstractCollection` ：

```java
// 省略其他
#28 = Methodref    #16.#114 // java/util/AbstractCollection.add:(Ljava/lang/Object;)Z
```

会发现这里 **“明明调用的就是 AbstractCollection.add 函数啊”，困惑 ing…**

其实这正是本专栏为什么要一直强调：**“是什么，为什么比怎么做更重要”** 这个思想。

因为通过对 `invokespecial` “是什么” 的学习，我们就知道 ** 多态函数会根据实际调用的对象类型运行时选择方法，在编译器无法确定。

那么，**为啥编译时给出函数签名是 `AbstractCollection.add` 呢**？

其实 #28 这个参数代表的符号引用提供了调用所需的方法名称，参数列表，参数类型和方法返回值等

如果没有这个参数，虚拟机只能找到类型，而不知道到底调用哪个方法。让它多为难？

此时，虚拟机默默地长叹一声，“做虚拟机好难…”



##### 3.6 “官方” 解读

关于第二部分描述的问题，《Effective Java》 给出的解释是：

问题出在 `HashSet` 的内部实现上， `addAll` 方法是基于 `add` 方法来实现的，虽然 HashSet 的文档并没有专门强调这一细节，但是这样做也是非常合理的。

`InstrumentedHashSet` 中的 `addAll` 方法首先给 `addCount` 加 3 ，然后利用 supper.addAll 来调用 HashSet 的 addAll 函数，然后调用被 `InstrumentedHashSet` 覆盖的 `add` 方法，每个元素调用一次，所以又加了 3 此，最终结果是 6。

很多人可能不会亲自分析，看到这一小段就会就此止步，认为自己 “真的懂了”。其实这就是很多人读了很多书时对很多知识一知半解，理解不够透彻的重要原因之一。

通过上面几步的分析之后，再看书本的解释就能理解地非常深刻。



#### 4. 总结

本文灵活运用本专栏介绍的几个核心方法和思想来学习和研究《Effective Java》中的一个典型问题。

本小节想向大家传达的核心思想是：

- **“是什么”，“为什么” 有时候比 “怎么做” 更重要**。很多时候搞懂了 “是什么”，“为什么”，你就知道该 “怎么做” 了。很多时候是因为我们只记忆了 “是什么” 和 “怎么做”，没有思考将两者之间联系起来的 “为什么”，才导致我们知其然，而不知其所以然。
- **猜想和验证是非常重要的学习方式**，希望大家在学习和工作中多做这种训练，这样才能更容易地发现自己的问题，让自己错误的理解得到纠正，才能让自己的思考更严谨和深入。
- **方法是通用的**，一个好方法往往能够解决至少一类问题，本专栏所分享的方法绝不仅限于学习《手册》，还用来学习 Java 相关的知识。希望大家帮这些方法当作帮助自己解决问题的习惯，遇到问题时信手拈来。
- ** 解决问题的方法不止有一种，** 而能够用什么方法解决问题取决于你的知识面。通过单纯的看书，通过自己动手写例子，通过源码调试，通过反汇编等不同方法所能够掌握和理解的程度显然是不一样的。这是造成不同人学习能力差距的重要原因之一，然而介绍这些内容的专栏极少，本专栏就是其中之一，然而很多人更喜欢 “买椟还珠”，不愿意在这方面下功夫。
- **每一个疑问背后可能都隐藏着至少一个知识盲区，隐藏着一个彻底搞懂某个知识的机会**，然而很多人总是忽视这种机会。
- **很多人正是因为平时有时间时不想 “浪费时间” 去研究问题，才会在真正需要某个知识时，浪费了更多的时间去解决问题，走更多的弯路**。

很多人学的不好，不光是读书读的少，而是没有 GET 到重点，没有掌握学习的方法。

“知道” 和 “理解” 是两回事，大家都知道 “授人以鱼不如授人以渔” 的道理，然而现实却是很多人根本不重视 “渔”，只重视找 “鱼”；大家都知道 “磨刀不误砍柴工” 的道理，然而现实生活中很多朋友着急 “砍柴”，很少 “磨刀”，认为 “磨刀” 是浪费时间。

然而，对同一个问题的认知可以分为好几个层次，很多人会在不同的层次认为自己 “懂了”，而不再往下深挖。

希望更多的人，能够从更深的层次来理解知识，而不只是记忆知识，这样才能够知其所以然，真正掌握和运用知识。

希望大家在未来的学习过程中，能够重视方法的价值，能够多一些思考，在未来的学习和工作中少走一些弯路。

如果你觉得本专栏对你有帮助，欢迎推荐给更多朋友，一起交流学习。



#### 5. 思考题

1、《Effective Java》该章节给出的自定义 `addAll()` 函数除了书中提到问题，还存在哪些隐患？

2、由于我们无法修改 JDK 源码库，我这里给出了类似的代码范例，请修改注释处下面一行代码，让 `Parent` 类中的 `eatAll` 函数，调用自身的 `eat` 函数，而不是子类的 `eat` 函数。

```java
public class Demo {
    public static void main(String[] args) {
        Child child = new Child();
        child.eatAll(Arrays.asList("a", "b"));
    }
}
import java.util.Collection;
import java.util.Iterator;

public class Parent<E> {

    public void eat(E e) {
        System.out.println("P:eat-->" + e);
    }
    public void eatAll(Collection<? extends E> c) {
        System.out.println("P:eatAll");
        Iterator<? extends E> iterator = c.iterator();
        while (iterator.hasNext()) {
            // 修改下面这行代码
           eat(iterator.next());
        }
    }
}
import java.util.Collection;

public class Child<E> extends Parent<E> {

    @Override
    public void eat(E e) {
        System.out.println("C:eat" + e);
    }

    @Override
    public void eatAll(Collection<? extends E> c) {
        System.out.println("C:eatAll");
        super.eatAll(c);
    }
}
```



## 作图篇



### 我们一起学思维导图



#### 1. 前言

《手册》的 “设计规约” 章节，针对不同的情况分别建议使用：用例图、状态图、时序图、类图活动图等各种 UML 图形。这些 UML 图形都是需求分析和系统设计强有力的工具。

**然而为何本节先从思维导图讲起呢？**

因为思维导图不仅是我们实际开发过程中进行需求梳理一种常见方式，更是我们学习和总结知识的强有力的工具。[1](https://www.imooc.com/read/55/article/1165#fn1)

但很多人画思维导图只不过是随手列举或者按照书目章节进行记录，不清楚思维导图应该记录什么内容。

各种内容如何更好地组织在一起，没有充分系统的画图理论，导致画图后很少再去看，画图后印象不深刻。

因此本节先介绍思维导图的相关知识和用法，在后续章节中再介绍各种常见 UML 图的概念和应用。



#### 2. 思维导图是什么？为什么要用它？



##### 2.1 什么是思维导图

我们首先看下维基百科关于思维导图的定义：

> A **mind map** is a diagram used to visually organize information. A mind map is hierarchical and shows relationships among pieces of the whole.

也就是说思维导图是一种可视化的图形思维工具，思维导图通过层级化的方式展示部分和整体的关系。

因此思维导图的核心之一就是拆分，即将整体合理地拆分成多个部分，以便于理解。



##### 2.2 思维导图解决什么问题？

思维导图可以帮助我们从多个角度思考问题；可以帮助我们理清复杂问题的逻辑关系，形成结构化思维。

平时学习知识时更多地是侧重于记忆，相当于拿答案去看题目，总会有中看着就会，实际运用时就忘的感觉。而思维导图则是关键词为导向，通过一个关键词在头脑中提取与其相关的所有案例，并将其归纳总结到思维导图中。通过思维导图构建繁杂的信息和知识网络的关系。

学习观通过机器学习反思人类学习的方法，其中讲到学习需要明确输入和输出，需要通过例子重塑信息和知识的关系（即理解知识），理清关系和拆分知识，验证知识的有效性。

思维导图是学习的一个重要手段，因为思维导图则满足学习的几个重要条件：
![图片描述](img/5dfc28a40001763b07160457.png)

（此图来源自《学习观》 ）

正如图上思维导图所示：借助思维导图我们可以明确问题的输入和输出；可以通过思维导图对知识进行拆分，可以通过归纳的方法去总结知识，也可以通过演绎的方法去运用知识，还可以通过思维导图理清各种知识之间的逻辑关系；可以通过思维导图了解学习内容的本质，通过分而治之的方式将复杂的问题简单化。思维导图的同级知识之间有相互独立性。

在实际的学习和开发过程中，我通常会使用思维导图来浓缩阅读图书的重点信息；使用思维导图来汇总关键的知识点相关信息；使用思维导图来总结编程的主要思想等。思维导图已经成为学习知识和分析问题的一个重要工具。



##### 2.3 思维导图的误区

**很多人因为不会用思维导图，而认为思维导图 “没有用”。**

**画思维导图的常见误区**有很多种，最常见如下图所示：

![图片描述](img/5dfc28c200019cdb13900688.png)
思维导图并不是越大越详细越好，思维导图的一个主要目的是压缩知识。

思维导图的知识点之间如果不独立，则压缩知识就不够充分，知识之间的关系就不够明确。

很多人读了很多书，学了很多 “知识” 却不会运用，其本质在于只是见过、记忆过，但是没有真正理解知识。

如果思维导图的知识之间毫无逻辑关系，没有太大意义，无法帮助我们理解知识，无法发挥出思维导图的优势。

思维导图的一大作用就是帮助我们理解知识，寻找知识的关联，形成知识网络，帮助我们加深印象。

随着学习的不断深入，之前的思维导图可能存在错误、遗漏等情况，这就需要对思维导图进行更新，然而很多人即使有新的理解也不愿意去更新思维导图，这样思维导图的价值就会非常有限。



#### 3. 画思维导图



##### 3.1 思维导图工具

俗话说：“工欲善其事必先利其器”。画思维导图亦是如此。[2](https://www.imooc.com/read/55/article/1165#fn2)

最常见的思维导图工具有: xmind, MindManager、MindMaster、FreeMind、 iThoughts、百度脑图等。

其中最出名的当属老牌的思维导图工具 xmind，支持多平台，使用简单，主题丰富，功能强大。
![图片描述](img/5dfc28d9000154cb24961540.png)

如上图所示，以 xmind 为例，**思维导图的核心组件**有：Topic (主题)、Subtopic（子主题）、RelationShip (联系)、Summary（概括）、Boundary（边界）、Notes （笔记）。

其中 主题和子主图是思维导图的核心；边界用来区分不同的概念领域；联系主要用来描述跨层知识之间的关系；概括则将多个知识点汇总为一个；笔记主要用来描述知识点。



##### 3.2 画思维导图的依据

回归概念，思维导图是一种可视化的思维方式，因此画思维导图的目的是帮助我们学习和思考。

设计思维导图的主要依据也就是如何学习的依据即：**明确输入输出、将信息压缩为知识、通过例子重塑大脑连接、利用逻辑拆分知识**。

只有明确输入和输出才能做到 “以终为始”，保证不跑题。

思维导图是知识的可视化，因此应该选取重点知识，从而压缩知识；思维导图的本质是帮助我们理解知识，因此要通过例子来验证输入和输出的关系；知识点之间要尽可能独立，而知识点之间又必然有某些关联，如分类、回归、组合、执行步骤等。

画思维导图时可以和 5w1h 分析法结合在一起，即思考问题是什么 (what) ？为什么 (why) ? 何人 (who) ？何时 (when)？何地 (where)？如何做 (how)。



##### 3.3 画思维导图的例子



###### 3.3.1 归纳法和演绎法

在学习和工作中，可以将共性的知识点总结在一起，如化整为零、化零为整、时间换空间，空间换时间，问题转化、加中间者、通用方案等，这就是在积累经验，理解知识的过程。

当设计方案遇到一些难题时，可以从归纳的知识和案例中得到启发，从而能够做到理论和实际相结合。

**通过这种学习的方式，知识就像滚雪球一样越滚越大，知识之间的联系越来越密切，越来越能够融会贯通，理解会越来越好。**

如问题转换部分示例如下：
![图片描述](img/5dfc28ee0001a7ec15001152.png)

再如有计算机科学领域流行一句话：

> All problems in computer science can be solved by another level of indirection.
>
> 计算机科学的所有问题都可以通过增加中间层来解决。

因此学习时将中间环节或第三方的场景梳理在一起：
![图片描述](img/5dfc29090001879b17081612.png)

随着我们读书越来越多，工作中遇到的情况越来越复杂，可以将更多案例添加到此思维导图中，这样就通过一个知识点将不同的知识点串联在了一起。

我们学习技术时还可以结合生活、哲学等，通过思维导图进行总结：
![图片描述](img/5dfc291d0001bd5416441814.png)

通过这种方式就实现了跨学科，跨科目，跨技术的学习，将知识学活。

有些人使用思维导图知识帮助自己 “记忆” 某些具体知识，这有些买椟还珠。

我们不需要一字不差地记忆右侧的某个具体知识点，我们需要的是通过右侧相似的具体案例，归纳出本质的规律，找到根本原因，加深对规律的理解，然后在使用时将其运用出去解决问题。

**学习的目的是学以致用，所谓学就是看书，思考等，用就是将学到的知识用来解决新的问题。**

通过上述方式构造知识之间的关联，加深对知识的理解，降低记忆难度。当遇到类的情况是更容易想到该知识点，更容易逆向运用到解决新的问题上。



###### 3.3.2 思考

可以使用思维导图总结学习的方法，比如总结如何学习，其中就包括写示例代码、画 UML 图、看官方手册、看技术网站、读源码、模拟设计、调试、抓包、反汇编等。

其中读源码部分的思维导图如下：
![图片描述](img/5dfc29370001711e10941436.png)

在不断学习进阶的过程中需要对该思维导图不断丰富，如果我们发现 “读一些源码分析的书” 是一个非常好的方法，可以将其添加到上述思维导图中。



###### 3.3.3 其他

我们可以使用思维导图梳理产品需求的核心要点和注意事项；总结从需求分析到项目上线过程中需要注意的问题；还可以通过思维导图梳理工作中常用的命令，总结学习和进阶必备的图书等。

这种相对比较简单，而且场景不同设计的思维导图内容差异很大，大家根据实际情况去画图即可。



#### 4. 总结

本节主要讲述什么是思维导图，为什么要使用思维导图，画思维导图有哪些依据和误区，有哪些思维导图的工具，并给出了自己的几个使用思维导图的范例。

俗话说：“纸上得来终觉浅，绝知此事要躬行”，希望大家可以通过本节的学习，能够重视思维导图，能够理解并熟练的运用思维导图，积累工作经验，深入理解知识，提高进阶的速率。

下一节将讲解流程图。



#### 5. 课后练习

结合自己的学习和工作经验，使用演绎的思想，整理 “化整为零”、“化零为整” 思想的思维导图。



### 我们一起学基本流程图



#### 1. 前言

能够学习和使用助于分析和解决问题的各种图形是计算机专业的专业素养体现。

前面讲了思维导图，倡导大家使用思维导图来学习知识，深化对知识的理解，并能够举一反三融会贯通。

在正式学习 UML 图之前，我们在本节学习做技术方案时常用的图形之一的**流程图**。



#### 2.What？Why?When?



##### 2.1 什么是流程图？

流程图是表示算法和计算机程序的工具，但现在已经被应用到各种流程中。

维基百科对流程图做出下面的描述 [1](https://www.imooc.com/read/55/article/1166#fn1)：

> 流程图 （Flow Chart）是表示算法、工作流或流程的一种框图表示，它以不同类型的框表示不同种类的步骤，两个步骤之间通过箭头连接。



##### 2.2 为什么要使用流程图？

流程图在信息展示中扮演着重要角色，它能够帮助我们将复杂的问题可视化，可以清楚地描述问题或任务的结构。流程图便于说明解决已知问题的方法，在分析、设计、记录及操控等很多领域的流程或程序中都有广泛应用。

在实际开发中，**流程图是程序员设计技术方案，开发人员之间交流设计思想的一个主要图形之一**。在交流技术问题时，很多优秀的程序员也会习惯性地，画流程图来交流思路。

同时流程图也是我们开发人员和产品、测试沟通的一个工具，他们可能看不懂你的代码，但是看得懂流程图。



##### 2.3 何时使用流程图？

**那么使用流程图的典型场景有哪些？**

- 当需要描述复杂流程时；
- 当分析一些不必要的步骤时；
- 当我们需要帮助团队成员理解程序时；
- 当我们需要设计一个新的流程时。



#### 3. How?

了解了流程图是什么，为什么需要流程图以及流程图的主要场景，接下来我们将学习如何画流程图。



##### 3.1 画图工具

还是那句话：“工欲善其事必先利其器”[2](https://www.imooc.com/read/55/article/1166#fn2)。

在学习和工作中接触了很多画流程图工具 [3](https://www.imooc.com/read/55/article/1166#fn3)，其中比较有名的有 Microsoft Office Visio[4](https://www.imooc.com/read/55/article/1166#fn4)、Visual Paradigm、Process On、亿图图示等。

本小节将采用 Visual Paradigm 来学习 流程图。



##### 3.2 核心符号

**终端符号**，表示流程的开始和结束：

![图片描述](img/5dfc34f700013b9901280062.png)
**流程**，表示某个特殊的操作。

![图片描述](img/5dfc35370001942501200062.png)
**文档**，表示一个输入或输出的介质，如输入或输出。
![图片描述](img/5dfc352a0001333e01000061.png)

**决策**，表示一个决策或分支点。采用菱形表示，表示此时分为多个情况，每种情况可表示一种子任务，通常有两种情况：是，否。
![图片描述](img/5dfc354f000177bb00840084.png)

**数据**，表示信息进入或离开处理程序。输入可能是一个外卖订单，输出可能是一个外卖包裹。
![图片描述](img/5dfc355e000129c401000062.png)

**延时或瓶颈**，表示程序的延时或者性能瓶颈。

![图片描述](img/5dfc35810001edf501000062.png)
**流**，表示流程处理的方向。

![图片描述](img/5dfc359f00012f8901340009.png)
**汇总**，表示多个流程汇总到一处。通常用在多个流指向同一个处理流程，为了美观，可以先指向该组件再从该组件指向该处理流程。

![图片描述](img/5dfc35be000151af01520152.png)
**或**，表示多个流程之间是或关系。

![图片描述](img/5dfc35d600016baf01540148.png)
**数据库**，表示文档和档案的存储。
![图片描述](img/5dfc35ea0001ecec01500138.png)

**页面内引用**，表示上一个或下一个步骤在此绘图的其他位置，需设置一串字符串来标识，多用于大型流程图。
![图片描述](img/5dfc35fd0001312e01360134.png)

**页面外引用**，表示将使用单独的页面绘制子流程图，需设置一串字符串来标识，多用于大型流程图。
![图片描述](img/5dfc36120001ba9401380136.png)



##### 3.3 画图步骤

首先明确画流程图是将复杂的任务拆分为多个流程，核心是子步骤的顺序与条件。

核心步骤分为以下 4 步：

1. 画一个只包含任务本身的流程图；
2. 画一个将任务拆分成多个子任务的流程图；
3. 考虑流程中可能出现的异常情况，在这些异常情况中添加决策节点和其它可能的路径；
4. 如果不能够清晰表达流程，则重复上述步骤。

对应的流程图如下：

![图片描述](img/5dfc363400013d2606961132.png)



##### 3.4 基本流程图举例

特别强调，画流程图的主要目的是让看此图的人能够清楚地表达执行流程，因此不必拘泥于细节。



###### 3.4.1 简单示例

按照上述画流程图的步骤，我们以用拖把拖地为例作图如下：
![图片描述](img/5dfc3646000196c907040703.png)



###### 3.4.2 模拟设计

比如产品让我们设计一个登录的功能，场景描述如下：

> 系统有两种登录方式，一种是通过手机号和短信验证码登录；一种是通过用户名密码登录。
>
> 如果是手机和短信验证码登录方式，用户点击获取验证码按钮，手机会收到验证码短信（60 秒内只能获取一次，验证码 5 分钟内有效），输入正确登录成功，输入错误提示失败。如果采用用户名密码登录，用户名和密码正确则登录成功，否则登录失败。

根据以上描述，我们可以作出下面的参考流程图：

![图片描述](img/5dfc365d0001a45209270799.png)

上面给出了两个简单的案例，希望大家在实际学习和开发过程中多加练习，熟能生巧。在实际工作中能够更快速准确地画出流程图，清楚地说明自己的意图。



#### 4. 总结

本节主要介绍了流程图的概念，为什么要使用流程图、何时使用流程图、以及作流程图的主要步骤，并给出了基本流程图的使用案例。

希望大家在设计技术方案时，在修改了某个功能提交提测文档时，在和其他开发人员沟通设计想法时，都可以尝试使用流程图，让自己的观点表达得更清楚。

下一节我们将一起学习用例图。



#### 5. 课后作业

回忆一下自己在 ATM 取款流程，通过流程图将取款流程画出来。



### 我们一起学用例图



#### 1. 前言

《手册》在 设计规约 部分对用例图有这样一条规定 [1](https://www.imooc.com/read/55/article/1167#fn1)：

> 【强制】在需求分析阶段，如果与系统交互的 User 超过一类并且相关的 User Case 超过 5 个，使用用例图来表达更清晰的结构化需求。

用例图是需求分析的一种重要图形工具，本小节我们将学习用例图的概念，用例图的核心组件，用例图的使用场景和使用案例等。



#### 2. 背景知识



##### 2.1 什么是 UML？为什么要使用 UML?

> UML 即统一建模语言，是一种用于说明、可视化、构建和编写面向对象、软件密集型系统的开放方法。 UML 对大规模、复杂系统的建模有极大的帮助。

UML 通过它的元模型和表示法，把通过文字和其他表达方法难以表达清楚的内容，简单直观的通过图形表达出来。

使用 UML 图可以让我们和客户，让软件开发的各个角色之间的沟通交流更加顺畅。实际开发中，我们画出各种 UML 图，前端、测试甚至产品都可以很容易地通过 UML 快速看明白我们的设计方案，这就是 UML 的价值所在。

就 Java 开发工程师而言，UML 图通常出现在技术文档中。通过 UML 图来表达我们的系统设计，帮助其他成员理解和评估我们的方案。另外这些需求梳理或者技术方案，对后面维护的同学有极大的参考价值。

可能会有很多同学会说：**我平时不画图也不影响开发啊？画图完全是在浪费时间。**

有这种认识的核心原因在于抵触心理，不敢离开舒适区；另外一点在于参与的项目都比较小，无法体会需求分析、系统复杂时 UML 图体现出来的价值。

很多朋友，尤其是新手，在需求分析阶段和方案设计阶段不重视，往往导致后期设计偏移需求，遗漏需求，甚至设计方案重新设计，部分代码重新编写等情况，反而浪费更多时间，付出更大的代价。

UML 图形主要分为结构型图形，行为型图形两大类。 UML 2.2 包含 14 中图，分别如下：
![图片描述](img/5dfc36d50001a9da07510516.png)

本专栏重点介绍工作中常用的：用例图、类图、时序图、状态机图和活动图。



##### 2.2 用例图是什么？为什么要使用用例图？

用例图是一种以**用户视角**来描述**系统功能**的图形。

用例图包括：系统（System）、用例（UseCase）、参与者（Actor）以及用例和参与者之间的关系。

其中被描述的事物就是系统；系统的参与者称为角色；角色在系统中要的做的事就是用例也叫行为。

用例图通常用在**需求分析**和**总体设计**阶段。**用例图的目的** 是让项目的参与者能够在更高层次上理解系统 [2](https://www.imooc.com/read/55/article/1167#fn2)。通过用例图可以基于用户视角对大型项目的功能进行拆分，而 **任务分解又是降低复杂度的核心方法之一**。



##### 2.3 核心组件



###### 2.3.1 基本组件

如下图所示，参与者使用小人图标表示，系统通过方形表示，用例一般采用椭圆形表示，参与者和用例之间通过连线来表示关系。
![图片描述](img/5dfc37090001fa3b08820486.png)
![用例图介绍](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsQAAA7EAZUrDhsAAAANSURBVBhXYzh8+PB/AAffA0nNPuCLAAAAAElFTkSuQmCC)

图 1 ：用例图的基本示例图



###### 2.3.2 关系描述

用例图中的关系主要是参与者和用例的关系，参与者和参与者的关系，用例和用例的关系为主。

其中参与者和用例的关系比较常见，如上图所示，角色 2 和行为 M 通过连线来表示他们之间的关系，即角色 2 使用该系统的目的之一或者该系统给角色 2 提供的功能之一就是行为 M。

**参与者和参与者之间**可能是并列的关系也可能是泛化关系，即面向对象语言中的继承关系。一般用户、管理员、超级管理员之间可以有继承关系。

**用例和用例之间**主要有 3 种关系，一种是包含关系（include）, 一种是拓展关系（extend），还有一种是继承关系。

其中**包含关系**描述一个用例需要某种功能，而该功能被定义为另外一个用例，使用带有 "<>" 的虚线箭头表示。如果管理学生信息，包括新增学生信息，修改学生信息，查询学生信息，删除学生信息。
![图片描述](img/5dfc37bf0001c92c11320678.png)
图 2：用例图包含关系示例

而**拓展关系**通常表示在某个用例的基础上，还能做什么事情，使用带有 "<>" 的虚线箭头表示。如下图所示，在读新闻的基础上，还支持打印新闻和分享新闻到朋友圈。
![图片描述](img/5dfc37de0001186509260406.png)

图 3：用例图拓展示例

大家可以回想一下 Java 中的继承，子类可以继承父类的属性和行为，且自己可以自定义自己的特有行为。而**用例的继承**与之类似。如下图所示，电视上投放广告具备一般投放广告行为，又有自己的特殊性，如支持视频；虽然广播的广告投放也属于投放广告的行为，但是通过音频方式传播；报纸的广告投放同样具备广告投放的功能，但是一般通过图片和文字来呈现。再如上一节讲到的登录，可以分为用户名密码登录和手机号短信登录，他们之间也是继承关系。
![图片描述](img/5dfc37fa0001a28809780556.png)

图 4 ：用例图的继承示例



#### 3. 示例



##### 3.1 画图工具

还是那句话：“工欲善其事必先利其器”。

**那么画各种 UML 图有哪些不错的工具呢？**

在学习和工作中接触了很多画流程图、UML 图工具，其中比较有名的有 PlantUML、Microsoft Office Visio、Visual Paradigm、StarUML、Process On、亿图图示等。

**我将这些画图软件分为两类，一类是文本语法类，一类是拖拽类**。

PlantUML 有自己的语法，它是一种 “文本型”，按照规定的语法可以快速作图，功能强大，支持时序图、用例图、类图、活动图、组件图、状态图、对象图、部署图和定时图等。PlantUML 的**主要优点**是：作图快速，文本存储方便修改，用户只需要关心内容，不需要过度关心样式，同时这也是它的**缺点**是输出的图片风格较为单一。

而其他软件则拖拽为主，优点是发挥的余地很大，缺点是作图效率相对较低，尤其是对于有些 “强迫症” 的小伙伴来说，简直是灾难，经常需要花费很多时间在各种内部组件和连线的对齐上。

因此如果无特殊喜好，画 UML 图形 个人最推荐 PlantUML。如果你喜欢拖拽， windows 系统用户建议使用 Visio 来画图， Mac 系统用户建议使用 Visual Paradigm。

本小节使用 Visual Paradigm 进行绘图。



##### 3.2 准备



###### 3.2.1 寻找参与者

想要画用例图，一个重要前置环节就是寻找参与者，参与者不仅包括人还可能包括系统。

首先提出几个问题：

- 系统给哪些人设计的？由哪些人使用？
- 系统由谁来负责维护？
- 系统由谁来管理？
- 系统为哪些人或系统提供数据？

这些问题都可以帮助我们快速找到用例。

如我们分析自动售卖机系统的用例：该系统是买东西给消费者，是设计给消费者用的；售卖机需要定期补货就需要货品管理人员；售卖机可能损坏，就需要更换或维修，因此需要运维人员的参与；售卖机的屏幕可以显示广告，因此可能吸引广告商来投放广告。



###### 3.2.2 确定用例

画用例图要明确有哪些用例，我们可以借助一系列提问来帮助我们明确用例：

- 参与者为什么要使用这个系统？
- 参与者是否有增删改查数据的行为？增删改查数据是谁来操作的？

另外我们还可以借助用例的特征来帮助我们判断用例是否正确：

**用例之间相互独立**。即每个用例不需要与其他用例交互就可以满足参与者的一个目的。用例从功能上是完整的，用例本质上体现了系统参与者的愿望，如果不能够完整表达参与者的愿望就不能称为用例。如去 ATM 机取钱，查询余额、取款等可以成为用例，而插入银行卡就不能称之为用例，因为它是其他用例的一个前置条件，而不是一个完整功能，也不是用户参与 ATM 机系统的目的。

**用例的执行结果对参与者来说是可观测和有意义的。** 即使有些功能时系统不可或缺的一部分，但是对用户而言是不可观测的，在需求分析阶段也不应该作为用例出现。另外作为一个单独事件对用户而言是无意义的也不应该成为用例，比如上节讲到的用户登录功能，那么登录可以称之为一个用例，而填写用户名、填写密码等就不能称之为用例，因为用户的核心目的是登录，单纯地填写用户名或者填写密码对用户而言是无意义，它们只是实现登录功能的一个重要步骤。

**用例必须由参与者发起，不应该自动启动，也不应该没有参与者，更不应该主动启动另外一个用例**。这也为我们寻找用例提供了一个突破口，当我们找全参与者后，可以围绕参与者的目的来推出用例。如商品售卖系统的用例包括消费者，消费则使用该系统的目的是购买商品；用例还包括运维人员，他们主要负责贩卖机的维修；以此类推，可以将相关的用例都推出来。

**用例必然是动宾结构**。即用例是由动作和目标构成。如取款、转账、注册账户、退出系统等。



###### 3.2.3 确定关系

确定好参与者和用例后，就要思考参与者之间，参与者和用例之间，用例和用例之间的关系。

梳理好这些关系以后，在作图时使用 2.3 所提供的图形绘制对应的关系。



##### 2.3 作图

某产品给出了一个需求描述：

> 我们要设计一个新闻系统，这个系统为了满足 xxxx 的需求，在这个系统中普通用户可以登录，读新闻；管理员除了具备一般用户的功能外，还可以写新闻、审核新闻，发布新闻；高级管理员具有以上所有功能外，还可以删除新闻。
>
> 普通用户不仅可以读新闻，将新闻内内容打印并可以支持分享到朋友圈，满足用户的个性化需求，增强用户的体验，提高互动度等。
>
> 管理员在审核新闻和发布新闻之前要可以预览新闻，避免审核和发布失误。
>
> 高级管理员在删除新闻之前要有弹窗提醒。

从上面的描述我们可知系统就是某新闻系统；参与者就有三类：普通用户、普通管理员、超级管理员；核心用例有登录、读新闻、写新闻、审核新闻、发布新闻、删除新闻等。

根据以上的分析，我们可以大致画图如下：
![图片描述](img/5dfc387d00010c3815160896.png)

图 ：某新闻系统用例图

画完此用例图后，我们重新对比需求，进行校对。如果发现该有却没有的功能，要分析是隐含在需求背后的功能，还是产品遗漏掉的功能，如果是重要的产品遗漏的功能，可以和产品反馈。比如我们发现退出系统的功能虽然产品没有给出，但是应该要有的功能，我们需要后续补上去。

在此要特别提醒大家，这需求分析时发现产品设计可能不合理的地方，可能遗漏的地方要及时沟通反馈。很多产品后期新增的需求，都是早期遗漏的需求。如果我们能够帮助产品在需求分析阶段找出遗漏的需求，就尽可能的避免后期被加需求。我们要做一个会思考的程序员，而不是执行命令的机器。



#### 4. 总结

本节主要讲述了用例图的概念、目的，用例图的主要组成部分和对应图形，并给出了用例图的用法示例。希望大家可以掌握用例图，通过用例图从用户的视角来分析系统。更多高级用法推荐通过《大象：Thiking in UML》[3](https://www.imooc.com/read/55/article/1167#fn3)、《火球：UML 大战需求分析》[4](https://www.imooc.com/read/55/article/1167#fn4) 相关章节深入学习。

下一节我们将学习类图，了解类图的概念、为什么要使用类图、类图的基本用法等。



#### 5. 课后练习

相信大家都有去 ATM 机取钱的经历，请大家分析如果我们设计 ATM 系统，使用用例图进行需求分析，参与者有哪些？有哪些用例？自己动手画一画。



### 我们一起学类图



#### 1. 前言

上一节我们学了用例图，本节我们将介绍类图。

《手册》设计规约章节，对类图有下面的规定：

> 【强制】如果系统中模型类超过 5 个，并且存在复杂的依赖关系，使用功能类图来表达并明确类之间的关系。

那么我们来思考几个问题：

- 什么是类图？
- 我们为什么要使用类图？
- 该如何画类图呢？



#### 2. 什么是类图？为什么要使用类图？

类图是软件工程的统一建模语言中的一种静态结构图，该图描述系统的类和其它类和属性之间的关系。类图描述了面向对象系统中的类的结构，以及类之间的关系，类图中也会体现出约束，也会展示出类的属性。
![图片描述](img/5dfc62940001148c07260332.png)

图 1：类图示例（图片来自 Visual Paradigm）

面向对象编程语言就不得不提到类和接口，在大型系统中类的数量众多，正如《手册》所说，当核心类较多且相互之间存在依赖关系时，需要使用类图将这种关系清晰地表达出来。

类图通常用在软件生命周期中的**详细设计阶段**，另外学习一定不要教条，当我们想从类视角来学习和研究源码时，类图也会派上大用场，下文将详细说明这一点。



#### 3 如何作图？



##### 3.1 了解基本组件



###### 3.1.1 类图符号

如下图所示，类图中，类主要由三部分构成：类名部分、属性部分和操作部分，这三个部分通过横线隔开。
![图片描述](img/5dfc62b300015f4e03300123.png)

图 2：类图示例（图片来自 Visual Paradigm）

属性的表示形式为：“可见性 属性名称：属性类型”。 上图中的 - content : String，就等价于 Java 在源码中的：

```
private String content;
```

操作的表示形式为：“可见性 函数名：返回值类型”。



###### 3.1.2 关系

就类图而言，关系主要指类之间的联系，主要包括依赖关系、关联关系、聚合关系、组合关系和泛化关系。

**依赖关系**是五种关系中耦合最小的一种关系，类 A 要完成某种功能引用了类 B，那么 A 就依赖了 B，如 类 A 的成员函数的返回值、参数、局部变量或静态方法调用使用了 B 。依赖关系的生命周期较短，一般在方法调用期间存在。

![图片描述](img/5dfc62e4000153f004800132.png)
图 3：依赖关系

**关联主要分为两类：单向关联和双向关联。** 关联关系的生命周期更长，随着类的初始化而产生，随着对象的销毁而结束。

其中**单向关联**，用带箭头的实线表示。如下图表示左侧类用到类右侧的类。关联关系表示类比较强的依赖，且这种关系由一方来维护，如学生依赖老师，学生类中有一个老师的成员属性。
![图片描述](img/5dfc63030001919e04600132.png)

图 4：单向关联

**双向关联**如下图所示，通过实线连接，表示两个类相互引用。
![图片描述](img/5dfc63150001c88204600132.png)

图 5：双向关联

**聚合关系**使用实心线加空心菱形表示。聚合用来表示集体和个体的关系，是一种 "has-a" 的关系。例如班级和学生之间，公司和员工之间就存在聚合关系。
![图片描述](img/5dfc638a0001a3a006080157.png)

图 6：聚合关系

**组合关系**是关联关系的一种特例，表示一个整体和其组成部分的关系，是一种 "contains-a" 关系。如人作为一个整体，包括头部、腹部、腿部等组成。
![图片描述](img/5dfc63340001557a04740464.png)

图 7：组合关系

**泛化关系** 主要指类与类之间的继承关系以及类与接口之间的实现关系。以 Integer 为例，实现了 Comparable 接口，继承了 Number 类，而 Number 类又实现了 Serializable 接口，可用如下类图表示。实现接口通过待箭头的虚线表示，而继承类则通过带箭头的实现表示。
![图片描述](img/5dfc63b00001172004850564.png)

图 8：泛化

**关系需要通过标识来表示对应数量之间的关系**。其中 `1` 表示只有一个；`0...1` 表示 0 个或者 1 个；`*` 表示多个；`0..*` 表示 0 个或多个；`1..*` 表示 1 个或多个。例如一个班级有一个或多个学生，但是一个学生只能在一个班级上课，则可用下图表示。
![图片描述](img/5dfc647400013f8f04260132.png)

图：数量标识

讲完了类之间的联系，我们接下来讲一下可见性。

类图的可见性分为四类：公有 (public) 用、私有 (private)、受保护（protected）、包私有（package private），和 Java 的访问修饰符的含义一致。

其中 public 用 `+` 表示， private 用 `-`, protected 用 `#` 表示， package private 使用 `~` 表示；
![图片描述](img/5dfc64930001e9e602590210.png)

图 10：可见性（图片来自 Visual Paradigm）

注意不同的画图工具的最终呈现形式会有差异，有些会直接显示上述符号，有些会通过不同的图形符号表示（如 PlantUML），而有些则会通过锁的颜色等方式来表示（如 IDEA 自带的类图插件）。



##### 3.2 作图

通过以上的学习我们对类图的基本组件有了基本的认识，下面我们从大家熟悉的几个类来了解类图的画法。



###### 3.2.1 有源码的情况

下面讲解一个技巧，作为常用的开发工具 IDEA 支持根据源码生成类图。

可以选择某个类，右键–> Diagrams --> Show Diagram --> Java Class Diagrams 来自动根据源码绘制类图。

下面是 Number 的类图：
![图片描述](img/5dfc64cc000178a204120552.png)

图 11 : Number 类的类图

如果想了解其它类和它的关系，可以直接将其它类拖到该页面上。可以通过页面的菜单开启关闭类图是否包含属性、操作等。
![图片描述](img/5dfc64e3000140f606870233.png)

图 12 : 类图示例

通过 IDEA 根据源码生成类图，可以清晰地了解不同类之间的关系，清楚地了解目标类的各种属性和函数，对我们学习源码有极大地帮助。也是我们学习画类图的一个非常权威的参考资料。

如我们通过 IDEA 自带的类图工具绘制 fastjson 核心类之间关系的类图，可以通过全局的视角来观察核心类之间的关系，从先整体后局部的思维来学习源码。
![图片描述](img/5dfc64f800013eff12390664.png)

图 13 : fastjson 核心类的类图



###### 3.2.2 没有源码的情况

假如有这样一个场景：

> 一个交易系统中主要包括顾客、订单、订单详情、商品、支付等核心类。其中顾客的属性包括顾客的姓名、收货地址、联系方式和是否活跃。订单的主要属性包括订单的创建日期，订单详情、订单的状态、支付方式。订单的支付方式可以有多种，如信用卡、现金、支票、转账。订单的状态包括创建状态、海运中、投递完成和支付完成这几个状态。订单详情包括商品数量，税信息，订单详情类还要提供计算总金额和重量的功能。订单详情中包含商品，商品有重量和描述信息，商品对象中要提供获取商品重量的接口，还要提供根据重量获取价格的功能。

根据上述描述可以绘制出核心的类、属性和行为，然后再去梳理类之间的关系。

类之间的关系需要通过需求或者常识得出。一个顾客可能未下单，即有零个订单，也可能多次下单从而产生多个订单。一个订单只有一种状态，因此订单和状态之间是一对一的关系。由于支持多种支付方式，因此订单和支付方式之间是一对多的关系。支付方式可以是一个抽象类，每种具体的支付方式是其实现类，因此它们之间是泛化关系。订单必然订单项，最少为一个。而一个商品有可能被一次或多次下单而成为订单项，也可能没有被下单过，因此一个商品可能对应零个或者多个订单订单项。

根据这个场景描述和分析就可以绘制出如下：
![图片描述](img/5dfc650e00010e6806670472.png)

图 1：类图示例（图片来自 Visual Paradigm）



#### 4. 总结

本节主要介绍了类图的基本概念和使用场景，介绍了类图的组件以及组件之间的关系并给出了相关范例。类图不仅是我们设计的工具，更是我们学习的工具，我们可以通过 IDEA 自带的类图工具来了解所要学习的源码核心列之间的关系，反向帮助我们理解源码。

下一节我们将讲述时序图的概念、使用场景和如何画时序图。



#### 5. 课后练习

使用 IDEA 自带的类图工具作图，通过类图来了解 Java 的集合体系。



### 我们一起学时序图



#### 1. 前言

前一节我们学习了类图，我们知道通过类图可以表示类之间的依赖、关联和泛型关系。

**那么如何表示类之间的调用路链呢？**

这就需要用到我们本节所要讲到的时序图。

《手册》设计规约章节，对时序图有下面的规定：

> 【强制】如果系统中某个功能的调用链路上的涉及对象超过 3 个，使用时序图来表达并且明
> 确各调用环节的输入与输出。

那么我们要思考几个问题：

- 什么是时序图？
- 时序图的使用场景有哪些？
- 如何画时序图？

接下来本节将重点研究这几个问题。



#### 2. 时序图是什么？为什么需要它？

**当你真正清楚一个概念，分析该概念和其他类似概念的区别之后，你就很容易知道为什么要用这个知识或者技术，在适合的场景才更容易想起来用它**。

接下来我们将学习时序图是什么？为什么需要用时序图？



##### 2.1 时序图的概念

时序图又称顺序图，属于 UML 行为图， 时序图主要用来表示对象之间的交互顺序。

> 时序图反映了一系列对象的交互与协作关系，清晰立体地反映系统的调用纵深链路。

时序图的核心元素包括：对象（Actor）、生命线（Lifeline）、控制焦点（Focus of control）、消息（Message）等。



##### 2.2 时序图的分类

根据《大象：Thinking in UML》中关于时序图的相关描述，我们可知，时序图使用场景分为三类：业务模型时序图、概念模型时序图和设计模型时序图。

**业务模型时序图**用于为领域模型中的业务实体交互建模，目标是实现业务用例。

**概念模型时序图**依据业务模型场景采用分析类来重新绘制一遍，目标同样是实现业务用例。因为分析类本身代表了系统原型，所以这时的时序图已经有了实现的影子。

**设计模型时序图**使用设计类作为对象绘制，目标是实现概念模型中的某个事件流，一般以一个完整交互为单位，消息细致到方法级别。因为设计模型时序图工作量实在太大，不需要为每一个交互都绘制时序图，但一定要有足够的概念模型时序图来支撑需求与实现之间的过渡。



##### 2.2 为什么要用时序图？

我们可以将时序图和其它 UML 图进行对比，来理解这个问题。

在需求分析和软件设计阶段，用例图、类图、时序图、活动图等使用比较多。

如果我们想表示多个对象之间的（时间）顺序，表示多个类之间的调用链路，那么用来表示参与者或系统之间关系的用例图显然不适合；而描述类的属性和方法类图以及类之间的依赖、关联和泛型关系的类图也显得力不从心；而活动图更侧重活动的流转，视角完全不同。因此就需要有一种图形可以从粗和细的粒度上表达表示用例之间的顺序关系，这就是时序图出现的重要原因之一。

**这里也说明了脱离场景无法谈好坏。**

脱离某个具体场景，你敢说用例图或者时序图就是 UML 图里最好的吗？如果真的是为啥还需要那么多种 UML 图呢？

**真正的牛人不仅仅是某个具体知识学得很透彻，更是能够根据特定场景选择最适合的方案的人。**



#### 3. 画时序图的准备



##### 3.1 了解基本组件

如下图所示，时序图的核心元素包括：参与者（Actor）、生命线（Lifeline）、控制焦点（Focus of control）、消息（Message）等。
![图片描述](img/5dfc656700017f5f08220272.png)

图 1 ： 时序图介绍



###### 3.1.1 生命线

**生命线** 是一条垂直的虚线，用来表示交互的独立个体，表示序列图中的对象在一段时间内的存在。

![图片描述](img/5dfc65900001896800710195.png)



###### 3.1.2 参与者

**参与者**，可以指人，外部其他系统，还可以指子系统。
![图片描述](img/5dfc65a70001a41700290195.png)



###### 3.1.3 控制焦点

**控制焦点** 表示对象执行一项操作的时期，是时序图中表示时间段的符号，用覆盖在生命线长矩形表示，矩形的顶部和箭头对齐，分别表示开始和结束时间，因此矩形的长度也表示持续的时间。
![图片描述](img/5dfc65c20001b02d01010151.png)



###### 3.1.4 消息

消息（Message）是对象之间的一种通信机制。

通常为了提高可读性，时序图的第一个消息总是从顶端开始，一般位于图的左上角。

#### 调用消息（Call Message）

调用消息对目标生命线的一次调用。通常使用实心箭头表示同步调用，使用左右朝向的开放箭头表示异步调用。
![图片描述](img/5dfc65ec0001ae2002110060.png)

#### 返回消息（Return Message）

返回消息表示目标对象传递给调用者的消息。使用朝向调用者的虚线开放箭头表示。
![图片描述](img/5dfc66010001513702110055.png)

#### 自调用消息（Self Message）

表示对当前生命线的调用消息，相当于一个对象的 A 函数调用该对象的 B 函数。
![图片描述](img/5dfc66180001d5a400350100.png)

#### 递归消息（Recursive Message）

递归消息表示对当前生命线的调用消息，相当于一个对象的 A 函数内部再次调用 A 函数。
![图片描述](img/5dfc66410001e08400350088.png)

#### 创建消息（Create Message）

创建消息表示目标生命的实例化消息，即初始化一个对象。使用朝向初始化对象的带虚线开放箭头表示。
![图片描述](img/5dfc666d0001f0cc02450124.png)

#### 销毁消息（Destory Message）

销毁消息表示破坏目标对象生命周期的请求。使用叉号结束生命线。
![图片描述](img/5dfc668a0001743f02190100.png)

#### 持续消息（Duration Message）

持续消息显示消息调用的两点之间的距离。

![图片描述](img/5dfc66c10001293f02140116.png)



###### 3.1.5 注释

可以将注释附着在各种元素上，注释不包括时序的语义，但可能包含对建模非常有用的信息。
![图片描述](img/5dfc66dd00012f8b00810041.png)



##### 3.2 消息和控制焦点

事件是指发生事情的交互的任意一点。

控制焦点也称为执行的发生，它是生命线上的一个窄的长的矩形。

如下图所示，同步消息箭头起始位置为消息开始事件，箭头所指向的位置为消息结束事件。控制焦点的顶端表示执行开始事件，底端表示执行结束事件。
![图片描述](img/5dfc66f60001b5a006100261.png)



##### 3.3 序列片段（Sequence Fragments）

UML 2.0 引入了序列片段，通过它可以更轻松地精确创建和维护时序图。

序列片段也成为组合片段，使用一个框来表示，它包含时序图的一部分交互。

序列片段左上角的符号表示片段的操作类型。

片段的主要类型包括： ref, assert, loop, break, alt, opt, neg。

![图片描述](img/5dfc670f0001a07404070215.png)

| 片段操作类型 | 片段类型介绍                             |
| :----------- | :--------------------------------------- |
| alt          | 只执行条件为真的片段                     |
| opt          | 可选，仅在条件为真时才执行               |
| par          | 并行：每个片段并行执行                   |
| loop         | 循环：指此片段可以多次执行               |
| region       | 关键区（临界区）：一次只能有一个线程执行 |
| neg          | 否定：片段显示无效的交互                 |
| ref          | 参考：指在另一个图上定义的交互           |
| sd           | 时序图：用于包围整个时序图               |



#### 4. 画时序图



##### 4.1 画图一般步骤

- 确定交互的上下文（即场景，如创建订单、购买商品、退货等）
- 识别参与过程的交互对象
- 为每个对象创建生命线
- 从初始消息开始，依次画出后续消息
- 为了清晰起见，调你家所需的返回消息
- 结合上述序列片段的几种操作类型，考虑消息的复杂逻辑
- 对时序图进行修改、美化



##### 4.2 示例

为了更好地说明时序图的优势，我先用文字描述一遍，再用时序图描述一遍同一件事，大家自行对感受一下。

开发中遇到一个异步同步数据导致旧值覆盖新值 BUG，分别使用文字和时序图对该问题描述。

文字描述：

> 发表朋友圈时可以选择文本或图片，如果只有文本不需要风控信息，如果发图片则需要检查风控信息。
>
> 点评状态最初为初始化状态，查询风控信息后更新为通过或者风控状态。
>
> 由于方案设计的不合理，在初始状态时存储一份，如果是文本则直接通过，如果是图片则查询风控信息后再存储一次。
>
> 而且为了查询性能，支持复杂的数据结构，支持超长度文本类型，支持丰富的查询需求，发送朋友圈状态数据存储到本地数据库后，还要异步通知存储服务，存储服务反查该朋友圈动态的信息存储一份 。
>
> 由于两次保存间隔极短，而通知存储服务反查是异步的，两次消息的顺序性没法保证，而且即使消息顺序抵达，未必先抵达的消息可以先处理，优先存储到 Elstic 中，所以可能存在 Elstic 旧值覆盖新值的情况，出现 BUG。

为了更清楚地描述该 BUG ，我们画出对应的时序图：
![图片描述](img/5dfc672a000141bb14421712.png)



#### 4.3 其它



##### 4.3.1 根据插件学时序图

大家还可以使用 IDEA 插件：[SequenceDiagram ](https://plugins.jetbrains.com/plugin/8286-sequencediagram/)来学习时序图，它可以根据代码调用链路自动生成时序图。

![图片描述](img/5dfc673f00014fb216000894.png)

该插件还支持点击生命线顶部的类名，点击请求的函数名称跳转到对应函数中。

这对我们学习时序图、熟悉自己项目代码和学习开源代码都有极大的帮助。



###### 4.3.2 根据案例学时序图

visual paradigm 给出了大量的[时序图范例](https://online.visual-paradigm.com/diagrams/examples/sequence-diagram/)，大家可以参考学习。
![图片描述](img/5dfc67530001a3dd08180585.png)



#### 5. 总结

本节主要学习了时序图的定义，时序图的使用场景，时序图的核心组件和画图步骤，并给出了一个范例。

介绍了两种高效学习时序图的方式，即使用时序图插件和看时序图案例学时序图，希望大家多学习多实践。

下一节我们将学习状态机图。



#### 6. 课后练习

回忆一下自己去 ATM 机取款的步骤，画出对应的时序图。



### 我们一起学状态机图



#### 1. 前言

前一节我们学到了用来表示对象交互顺序的时序图，那么有没有图形可以表示对象的状态变化呢？

答案是：有，它就是**状态机图**。

《手册》的设计规约章节有对状态图的规定：

> 【强制】如果某个业务对象的状态超过 3 个，使用状态图来表达并明确状态变化变化的各个触发条件。

那么我们思考下面几个问题：

- 什么是状态机图？
- 状态机图的使用场景是什么？
- 如何画状态机图？



#### 2. 状态机图是什么？使用场景有哪些？



##### 2.1 概念

状态机图也叫状态图、状态转换图、简单状态图，它描述一个事物随着事件的状态变化。

简而言之，**状态机图主要描述对象的状态以及引起对象状态变换的事件**。

状态机图可以对对象、用例甚至整个系统的行为建模。

大多数面向对象技术都适用状态机图来描述一个对象在其生命周期中的状态变化。



##### 2.2 使用场景

在业务建模时，可以创建状态机图对用例场景建模（和手册描述的情况类似）。

在分析和设计时，可以建模事件驱动对象；我们还可以使用多个状态机图来表现同一个状态机和行为的不同方面。



#### 3. 作图前的准备



##### 3.1 核心组件

为了更全面地描述核心组件的构成，本小节采用 visual paradigm 官网状态机图组件给大家讲解分析。



###### 3.1.1 状态

状态表示一个对象生命周期中的一种条件，它可以满足执行某些活动的条件后者等待接收某些事件。

状态包括 5 个部分：

- 状态名：状态的名称
- 进入（）：进入状态的行为
- 行为（do）：在进入状态时执行的行为
- 退出行为（）：离开状态时执行的操作
- 延迟触发（Deferrable Trigger）：暂时不触发状态变更的行为
  ![图片描述](img/5dfc679100018b5f06630208.png)

图 1 ：状态 （图片来自 visual paradigm）



###### 3.1.2 转换

转换是指前一个状态到后一个状态经历的事件。转换前的状态称为源状态，转变后的状态称为目标状态。

转化分为 5 个部分：

- 源状态：受变换影响的状态
- 事件触发器：一种可以触发源状态以满足保护条件的机制
- 保护条件：在接收事件触发器转换时需要计算的布尔表达式
- 操作：可执行的原子计算，可以直接作用于拥有状态机的对象，并间接作用于对象可见的其他对象
- 目标状态：转换完成后所处的状态
  ![图片描述](img/5dfc67b100017dad04600041.png)

图 2 ：转换 （图片来自 visual paradigm）



###### 3.1.3 决策节点

**分节点**（Fork node）是一种为状态，用来表示进入的转换将分为两个或者多个目标状态。从 fork 定点传入的流转不允许有守护条件或者触发器，至少有两个或两个以上的转出条件。

**合节点**（Join node）也是一种伪状态，合并从分支节点转换而来的多个状态，合节点至少有两个或两个以上的转入和一个转出。
![图片描述](img/5dfc67da000102ad05830175.png)

图 3 ：fork and join 节点 （图片来自 visual paradigm）

**选择**（Choice）是一种伪状态，通过守护条件判断执行的路径。
![图片描述](img/5dfc67ec0001feb801390168.png)

图 4：choice （图片来自 visual paradigm）

如下图所示下单时，如果库存足够则进入确认订单状态，如果库存不足或用户取消，则进入取消状态。
![图片描述](img/5dfc680d0001bbc903310139.png)

图 5：状态机图选择节点案例 （图片来自 visual paradigm）

**终止**

终止也是一种虚拟状态，表示状态机的生命周期结束。终止用待叉号的箭头表示。

和最终状态不同的是，终止状态表示由于上下文对象的结束而导致状态机的结束

如下图所示，断电导致激活状态转为终止状态。
![图片描述](img/5dfc68290001e10f02140041.png)

图 6：状态机图终止（图片来自 visual paradigm）



###### 3.1.4 组合状态

普通的状态是不包含子结构的，而组合状态则需要画在状态内部或者画到独立的状态图中，表示将一个状态分为多个状态。包含子状态的状态称之为组合状态。

如下图的 Active 状态就是一个组合状态，其中的 Inspection / Choice/ Transaction 只在 Active 状态下才存在。

需要注意的是，组合状态最多只有一个初始和最终状态。
![图片描述](img/5dfc6842000109cf06970296.png)

图 7：状态机图终止（图片来自 visual paradigm）

**组合状态和子状态机图**

下图为例，左侧表示组合状态，右侧表示子状态机图，两者语义上等价。
![图片描述](img/5dfc68550001ac1e05940224.png)

图 8：组合状态和子状态机图（图片来自 visual paradigm）

**正交状态**

如果一个组合状态包含两个及两个一样的子区域，则称之为正交。正交状态通过虚线分隔为多个部分。

如下图所示， S2 转换到状态 S1（转换到正交状态的边界）即代表激活了所有区域的初始状态。在正交状态中，每个区域都必须能够触达结束状态从而能够触发完成事件。 S3 转换到 S1 则表示并发的情况。
![图片描述](img/5dfc686600010e3307450304.png)

图 9：正交状态（图片来自 visual paradigm）

**历史状态**

历史状态允许状态机重新进入已经离开的组合状态。
![图片描述](img/5dfc687800019ac103730249.png)

图 10：历史状态（图片来自 visual paradigm）



##### 3.2 绘图步骤

**寻找主要状态**。画状态机图最重要的步骤是寻找主要状态。

**确定状态间的转换**，可以先将状态绘制出来，然后再分析状态之间的转换过程，绘制过程中可以调整状态的位置，以实现更好的视觉效果。

**细化状态内的活动和转换**。绘制完状态和状态间的转换后，可以根据需要添加内部转换，进入和退出转换，和其他的相关活动。

**组合和正交状态的使用**。如果某些状态可以归结为一个状态，那么可以使用组合状态绘制。如果一个状态可分为多个部分（转换流程），可以使用正交状态。



#### 4. 状态机图范例



##### 4.1 人生如梦

人的一生从婚姻角度看，包括单身、已婚、离异三个大状态；从职场角度看主要包括待业、受雇、退休三个大状态，当然从其他角度还可以画出更详细的状态图。

我们使用 [PlantUML](http://plantuml.com/zh/activity-diagram-beta)，从婚姻和求职角度画出人生的状态图。

> 注：大家可以在 IDEA 中安装 PlantUML 插件，在项目视图下找到 Scratches and Consoles 选项选项卡，在此目录下作图。

![图片描述](img/5dfc688e0001e06c16990535.png)
图 11：人生状态图



##### 4.2 评论审核状态图

另外以某个博客评论审核系统为例，演示状态图的画法。

> 某博客系统发表评论后需要审核，审核通过后读者才可见，审核不通过则需要重新修改，修改通过才能读者可见。

我们先寻找主要状态：待审核、待修改、公开，然后根据实际情况填充状态之间的转换。
![图片描述](img/5dfc68a60001efb911480582.png)

图 12：评论审核状态图



#### 5. 总结

本节主要讲述了状态机图的主要概念，常见的使用场景，核心的组件，并给出了使用范例。希望大家能够熟练掌握并运用到项目梳理和技术方案的设计中。

下一节将讲述活动图的概念，活动图和状态图的区别，以及活动图的画法等。



#### 6. 课后题

请参考本节人生的状态图，自行绘制一模一样的图形。将该图受雇状态细化为包括普通员工、公司主管、CTO 三个状态采用组合状态丰富人生的状态图。然后从年龄段的角度讲婴儿、少儿、青少年、青年、中年、老年补充到人生的状态图中。



### 我们一起学活动图



#### 1.前言

前一节我们学到了描述对象状态随着事件变化的状态机图，那么有没有一种图描述对象协作关系的图形呢？

答案是，有，那就是活动图。

> 【强制】如果系统中超过 2 个对象之间存在协作关系，并且需要表示复杂的处理流程，使用活动图来表示。
>
> 说明：活动图是流程图的扩展，增加了体现协作关系的对象泳道，支持表示并发等。

那么我们思考几个问题：

- 什么是活动图？
- 为什么要使用活动图？
- 如何画活动图呢？

本小节将为你分析这些问题。



#### 2.活动图是什么？它的使用场景有哪些？



##### 2.1 活动图的概念

活动图是工作流的图形表示，活动图主要由活动和动作构成，可以支持分支选择、迭代、并行。在 UML 中，活动图主要用于为计算性和组织性过程（即工作流）建模，相关活动之间的数据流也在其覆盖范围。

活动图显示连接动作和其他节点（如决策、分支、连接、合并和对象节点）的流。一般情况下活动和活动图之间是一一对应关系。

除非一个活动图表示一个连续的循环，否则，活动图应该有一个使活动开始的初始动作，还应该有一个或者多个终止动作。实心圆表示活动开始，牛眼符号表示活动结束。

流可以分支和合并。在活动图中用钻石表示分支条件，分支条件的出口由事件（如 Yes, No）或守护条件（如下图中的[order accepted]，[order rejectes]）来控制。
![图片描述](img/5dfc68dc0001814c09410399.png)

图 1：活动图描述（图片来自 visual paradigm）

流还可以分叉和再连接。这就产生了并发（并行）的计算线程。流的分叉和连接用短线表示。没有并发过程的流程图和传统的流程图非常相似。



##### 2.2 活动图和状态图关系

活动图简化了流程图并添加了一些新的符号。

状态图着重描述从一个状态到另一个状态的流程，主要有外部事件的参与。

活动图将流程分为一个一个活动，通过活动的先后顺序来展示流程；而状态机图从某个事物的状态是如何变化的角度来描述流程。



##### 2.3 使用场景

正如《手册》所说，当系统中超过 2 个对象之间存在协作关系，并且需要保湿复杂的处理流程时，需要使用活动图表示。



#### 3.绘图



##### 3.1 核心组件



###### 3.1.1 初始节点（Initial Node）

活动发生前的状态称之为开始状态。除非有嵌套活动，否则一个流程只能有一个开始状态。



###### 3.1.2 最终节点（Final Node）

最终节点采用大圆套小圆表示，其中内部的小圆为实心。活动图只能有一个初始状态，但是可以有 0 个或多个最终状态。
![图片描述](img/5dfc68f10001646a05840222.png)

图 2：活动图（图片来自 visual paradigm）



###### 3.1.3 流的最终节点

UML 2.0 新增了一个节点类型，称之为流的最终节点，用来代替活动的最终节点来表示流的终结。
![图片描述](img/5dfc6908000160cc05030145.png)

图 3：活动图最终节点（图片来自 visual paradigm）



###### 3.1.4 流程转换

活动图包含多种活动状态，那么这些状态之间通过什么关联呢？状态流转就应运而生了。

**状态流转包括控制流转和对象流转。**

**控制流转（Control Flow）**或状态流转也叫做路径或边。它用来表示从一个活动状态到另外一个活动状态的转变。我们使用带箭头的实线来表示。

![图片描述](img/5dfc691b0001d12c07290348.png)
图 4：流程转换（图片来自 visual paradigm）

**对象流转（Object Flow）**发生在活动和对象之间。一个活动状态使用对象作为输入，则从对象用箭头指向该活动状态。如果一个活动状态需要更新或者产生一个对象作为输出，那么箭头需要从该活动状态指向对象。



###### 3.1.5 决策节点和分支

**决策节点**

决策节点承接一个流程控制来源，将其拆分为多个流程控制出口。
![图片描述](img/5dfc693d0001af4001680113.png)

**合并节点**

合并节点将多个可选分支汇聚到一个节点中。
![图片描述](img/5dfc69560001080e01790085.png)

**Fork 节点**

Fork 节点将一个流程分成多个并发流程。
![图片描述](img/5dfc69690001459b01590133.png)

**Join 节点**

一个join 节点是同步多流程的控制节点。它有多个入口边，只有一个出口边。
![图片描述](img/5dfc69e20001379401620133.png)



###### 3.1.6 守护

活动图中，**守护**（Guard）是一种真假条件，决定状态的流转。



###### 3.1.7 对象节点

UML 2.0 的活动建模支持对象节点。
![图片描述](img/5dfc6a020001b05d04640043.png)



###### 3.1.8 数据存储

数据存储表示对象的持久化。
![图片描述](img/5dfc6a1e0001b6ff01690313.png)



###### 3.1.9 备注

备注支持为活动图中的元素进行注释，可以承载对建模有用的信息。
![图片描述](img/5dfc6a30000158c801860041.png)



###### 3.1.0 泳道

泳道表示不同的信息种类，将整个流程图结合到不同得到参与者视角中。

下图表示顾客在线采购超时商品并下单，销售商实地挑选商品并安排运输的泳道图。
![图片描述](img/5dfc6a4000012da103230362.png)



###### 3.1.11 时间事件和事件信号

**时间事件**表示活动的时间描述，采用水漏的图形表示。

下图表示每周三执行备份活动。
![图片描述](img/5dfc6a5900014c5a02530061.png)

**接收事件活动**（Accept Event Action）在活动图的业务建模中非常重要。它表示接收活动等待事件的发生。事件接收后，活动将被执行。

**发送信号活动**（Send Signal Action）表示接收活动作出反应的信号。

![图片描述](img/5dfc6a6a00012c1305520189.png)



##### 3.2 绘图步骤

构造结构图通常先为用例添加开始和结束点，为用例的主要步骤添加一个活动，从每个活动到其他活动、决策点和终点添加转换，在并行活动的地方添加同步条。

绘制活动图的主要步骤如下：

1. 首先，决定是否采用泳道，主要根据活动图中是否要体现出活动图的不同实施者；
2. 然后，尽量使用分支、分叉和汇合等基本的构建元素来描述活动控制流程；
3. 如果需要，加入对象流以及对象的状态变化；
4. 如果需要，使用高级建模元素（如辅助活动图、汇合描述、发送信号和接收信号和备注等）来表示更详细的信息。

回顾之前的几种图形的绘制过程，我们可以发现，**绘图的步骤基本都是先绘制主要信息再进行丰富，符合先整体后局部，先易后难的方式**。因此我们在绘图过程中，不需要背诵具体的绘图步骤，记住这个绘图原则即可。



##### 3.3 参考范例

上一节我们使用状态图绘制了某博客系统发表评论的步骤，本节将使用 [PlantUML](http://plantuml.com/zh/activity-diagram-beta) 绘制对应的活动图。

场景描述：

> 读者在某博客系统阅读文章后可以发表评论，但是评论需要作者审核，审核通过后对其他读者才可见，审核不通过则需要重新修改。

由于这里有两个角色，读者和作者，因此我们采用泳道的方式绘图。

评论审核是否通过需要走不通的流程，因此需要使用分支进行活动的流程控制。

根据场景描述以及上述分析，我们绘图如下：
![图片描述](img/5dfc6a7a0001fd2313651363.png)



#### 4.总结

本节主要介绍了活动图的概念、活动图的使用场景、活动图的核心组件，并给出了活动图的使用范例。希望大家可以结合 PlantUML中活动图的相关语法示例，结合 visual paradigm 的活动图范例，结合本节给出的绘图步骤，进行模仿绘图。



## 避坑篇



### Git和数据库篇



#### 1.前言

本章开始我们进入避坑篇，重点讲解开发中的相关坑点和一些避坑经验。

Git 和数据库是我们平时开发中常用到的技术，但是使用不当很容易出坑。

本小节将结合实际的开发经验，讲解 git 和 数据库相关比较有代表性的坑点和如何避坑[1](https://www.imooc.com/read/55/article/1172#fn1)。



#### 2.Git 相关



##### 2.1 相关教程、软件

本小节的重点不在于教大家如何学习 Git ，而是重点讲解实际开发中可能遇到问题。

本部分小介绍一些经典的 Git 资料，方便大家学习。



###### 2.1.1 资料

正如本专栏一直强调的：学习技术我们优先去官网看官方文档。

> 很多人正是因为看官方文档时没时间，才导致遇到 N 多问题浪费更多时间去解决，希望大家一定重视起来。

Git 官网提供了技术的介绍， Git 的下载地址，[官方文档](https://www.imooc.com/read/55/article/[https://git-scm.com/book/zh/v2/起步-关于版本控制](https://git-scm.com/book/zh/v2/起步-关于版本控制))，还非常贴心得给出了 [《Pro Git》 电子书](https://git-scm.com/book/zh/v2)。
![图片描述](img/5dfc5e030001c07a20841204.png)

大家还可以去 廖雪峰大大的官网上学习 [Git 相关用法](https://www.liaoxuefeng.com/wiki/896043488029600)。



###### 2.1.2 软件

如果你喜欢原生命令，直接敲命令即可。如果你更喜欢用软件可以用 IDEA 自带的 Git 工具，还可以用 [SourceTree](https://www.sourcetreeapp.com/) 、[Tower](https://www.git-tower.com/)等可视工具。
![图片描述](img/5dfc5e110001e6b028261672.png)

图：SourceTree 软件界面



##### 2.2 Git 相关的坑点

Git 的相关坑点主要是由于 Git 掌握不熟，还有粗心导致的。



###### 2.2.1 常见错误

案例1：检出错误

对于有错误提示的错误，一定要认真看错误提示的内容。

如下面的提示：

```
ssh: connect to host xxx.com port 22: Connection refused
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

显然要检查是否自己账户有权限或者自己clone 的地址是否有误。

案例2：切换邮箱提交错误

如下图所示，开发中可能还会遇到的一种错误是不小心切换了 git 账户，新的账户没有提交权限，然而你在新的账户上有了一次提交，此时代码推不上去，切换回原邮箱依然提示推不上去。肿么办…
![图片描述](img/5dfc5e330001222105420345.png)

图：某位同学遇到的类似问题图示

遇到这种问题大家一定要冷静思考，更换账户后为啥还推不上去呢？

通过 `git log` 可以看到提交历史中包含了之前错误账户的提交，因此解决问题的思路就是使用新的账户撤销错误那次提交。

其中一个方法如下：

1. 重新设置回原来的账户；
2. 回退到更换邮箱之前的版本：`git reset --soft d1a9eac6`；
3. 重新提交然后推送即可。

![图片描述](img/5dfc5e46000109e005960325.png)
执行上述步骤后，我们发现提交记录里已经没有 [wrong.qq.com](http://wrong.qq.com/) 这个错误账户的提交记录，因此重新推送代码就不会再报错。



###### 2.2.2 粗心类错误

案例1：切分支错误

比如你同时开发多个项目，你本应该按照下面的步骤基于master 拉出两个功能分支：

1. 基于 master 分支拉出了 feature-a 分支，来开发 A 功能；
2. 然后再从 master 分支拉出 feature-b 分支，来开发 B 功能。

但是由于粗心直接从 feature-a 拉出了 feature-b 分支。

最可怕的是，feature-b 的功能测试都通过且先上线。此时 feature-a 的代码就会被带到线上，极可能导致故障。

【建议】拉取新分支时一定要注意检查父分支。

案例2：提前合并到 master

有些功能还未到发布时间，提前一晚合并到 master ，由于晚上有其他团队成员紧急发布，导致自己的代码被带到了线上，然而线上相关的表DDL 还没提交，其他配置还没配置好。

从而发生故障，无故躺枪，杯具…

**案例3: merge 代码冲突解决不当**

如果两个人在同一个分支开发或者多个分支合并到主分支容易出现冲突，如果冲突解决不当可能造成故障。

比如冲突时没有了解清楚就删除了别人的代码，导致上线后出现问题。

【建议】 合并代码遇到冲突时，如果没有把握，联系和你冲突的人员一起确认。

**案例4：一个分支干了多件事**

有些人喜欢偷偷搞点小动作，某个功能专用的分支做了点其他事情，比如顺手优化了某个接口，却不告知测试人员。

导致测试没有覆盖到所有修改的场景，上线以后可能出现问题。

【建议】 一个分支只干一件事，如果顺带做了点其它改动一定要告知测试人员。



###### 2.2.3 其他建议

【建议】开发周期较长时，及时合并 master 分支解决冲突，避免最后阶段大量冲突，同时避免自己依赖的接口修改、删除和废弃等情况。

【建议】开发过程中要多将自己的开发分支和 master 代码进行比对，自我 Code Review，有助于及早发现问题。



##### 2.3 数据库相关



###### 2.3.1 案例分析

数据库本身的学习可以参考其他教程，本小节重点讲述实际开发中数据库相关的坑。

**案例1：没有唯一键约束**

《手册》中有涉及唯一键的描述：

> 【强制】业务上具有唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。

很多人会认为只要先查后插就能保证唯一，实则不然。

在高并发场景下，如果没有同步操作，两个事务同时开启，先后查询到没有数据，然后依次执行插入会出现重复数据。

是不是非高并发场景就没问题呢？结果也是否定的。

一方面别人写代码如果没有先查后插入而是直接插入，无法保证唯一性；另外一方面不是所有的数据都是通过 Java 代码插入到数据库的，如果通过数据平台写入时看到没有唯一键约束就没考虑数据重复的问题，极可能会写入重复的数据。

因此业务上具有唯一性的字段，即使是多个字段组合，也必须创建唯一键。

**案例2：执行 delete 等危险操作没带查询条件**

有些朋友手抖，执行delete 语句时不带条件或者删除条件有误，将可能造成事故。

这里给出一条避坑建议：

【建议】执行删除操作时，要开启事务，同时要先查询核对影响的行数和数据准确性，再执行删除操作 ；执行删除操作后再次核实，如果情况不对立即回滚。

**案例3：新数据或者功能没考虑对老数据的影响**

比如在没下旧接口的情况下要新增接口，而新接口要修改数据库表结构新增一个非 null 字段。

此时如果没考虑老接口，上线以后老接口插入数据时就会报错。

**案例4：时效性要求极高的场景查了备库**

有些接口对时效性要求较高，比如支付后还要通过另外一个接口查询支付结果，如果查询支付结果的接口走的是备库，此时恰有一个 DDL 导致主备延时较高，很容易查询出不符合预期的结果。

然而支付的结果影响较大，短暂的主备延迟可能会造成意想不到的后果。

因此给出以下建议：

【建议】大表的 DDL 不要在高峰期执行。

【建议】对于实时性要求极高的接口，请直接查询主库。

**案例5：MyBatis 映射语句问题**

一个比较典型的错误案例如下：

```sql
select * from xxx
where
<if test="a  != null">
 a = #{a}
</if>
<if test="b  != null">
 and b = #{b}
</if>
```

如果 a 和 b 同时为 null，则生成的语句将会是: `select * from xxx where` 从而出现语法错误。

此种情况应该使用 标签来代替 where 关键字。

另外一个典型的错误：

```sql
select * from students 
<where>
<if test="a  != null">
 a = #{a}
</if>
</where>
limit #{offset},#{limit}
```

此时，如果 offset 或者 limit 传入负数，就会报错。

> ERROR when execute SQL: SELECT * FROM students limit -1,10
> SyntaxError: Parse error on line 1:
> …FROM students limit -1,10
> -----------------------^
> Expecting ‘NUMBER’, got ‘MINUS’

因此，对于分页的参数一定要做好参数校验。

**案例6：悬停时间较长的事务被kill**

一个朋友在开发中遇到这样一种编码逻辑：查询出所有数据，然后循环根据每个数据的特征查询其她接口作为参数计算出值，然后插入到数据库，循环结束后提交事务。

之前数据量小时都没问题，数据量大以后该函数执行报错。

进行了各种分析，各种调试，各种尝试都没解决。

最终咨询 DBA 才得知根本原因是事务悬停超过 xxx 秒就会被 kill。

可以计算完成将结果每几百个一批批量插入到数据库；可以通过公司的大数据处理平台实现这个功能。

**案例7：表新增了供查询的字段，却没建索引，导致慢查询**

建表时很多人会想着创建索引加快查询速度。但是由于新增某个功能加新字段时，很容易忘记加索引。如果数据量较大，并发量较大时，容易导致慢查询。

【建议】新增字段如果作为查询条件时，要思考是否要加索引。



###### 2.3.2 选读

《手册》中有这样一个规定：【强制】页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决。

给出的说明是：索引文件具有 B-Tree 的最左前缀匹配特征，如果左侧的值未确定，那么无法使用此索引。

因此很多朋友可能会这么写：

```sql
select *  from xxx where name like CONCAT(#{name},'%') 
```

我们要思考一个问题：**这样写真的能避免左侧模糊查询吗？**

如果参数 name 的值为 ： “%” 或者 "%某字符"呢？

MySQL 中结果就等价于 ：

```sql
select *  from xxx where name like '%%' 
```

或者

```java
select *  from xxx where name like '%某字符%' 
```

依然不会走到索引。

因此对于 like 的参数，大家可以进行参数检查，也可以进行转义处理。



#### 3.总结

本节结合实际的开发经验讲述了 Git 和数据库中常见的坑点，并给出了一些建议。

希望大家在学习 Git 和数据库相关知识时，要注重思考，多动手实践去验证。

下一节将介绍其它各种坑点，给出对大家实际的学习开发有帮助的建议。



### 综合篇



#### 1. 前言

前面一小节集中讲述了 Git 和数据库相关的典型坑点并给出了一些建议。

本节将结合实际的开发经验给出其他各种坑点合集，以帮助大家尽可能地体会开发中可能会遇到的典型坑点，从而尽早避免 [1](https://www.imooc.com/read/55/article/1173#fn1)。



#### 2. 各种趟坑姿势

Java 工程师的进阶之路就是不断的踩坑之路。

聪明人和普通人的区别之一是，聪明人喜欢吸取别人的踩坑经验，而普通人不重视别人的经验，很多道理只有自己亲身经历，遭受挫折才开始重视。
![图片描述](img/5dfc5ea50001a8f216001066.png)图：好大的一个坑



##### 2.1 二方服务的坑

**案例 1：空指针异常**

在大一些的公司工作，不免要调用公司其他团队的服务接口，然而二方接口的很多返回值都可能为空。

往往在测试时对接的功能都很正常，结果上线以后，各种空指针异常，囧…

【建议】 写代码代码时一定要注意避免空指针。 具体的避免空指针方法参见第一章的对应小节。

**案例 2：幂等性问题**

有时候消息可能重复消费，你提供的服务对方也可能会因超时重试，如果没做好幂等，很可能出现重复下单、重复付款等严重的问题。

【建议】对于重复调用会有副作用的接口要注意保持接口的幂等。通常上游传入业务识别号和数据编号，作为幂等的依据，采用数据库唯一键等方式做幂等。

**案例 3：二方服务不可用**

二方服务不都是可靠的，偶尔可能会出现短暂的不可用。因此大家在和其他团队对接时，要考虑如果对方接口挂掉，我们该如何处理？

比如内容被风控是小概率事件，不应该阻碍核心功能。如果你负责对接风控，如果场景允许，当对方服务异常时可以做降级处理，直接返回风控通过，避免阻塞核心流程。

**案例 4：二方接口废弃**

我们在对接二方服务时，可能会发现自己找的接口或者对方提供的接口可能被标记废弃。

如果对方服务 jar 包提供了源码包，我们可以看其源码注释，查看是否有废弃的原因并给出替代的新接口，如果没有一定要和对方核实。

**案例 5：接口返回值为基本类型**

如果对接二方服务时，如果发现对方的接口中如果包含了基本类型的返回值。

因为基本类型的返回值都会有默认值，一定要确认这些默认值对自己的数据有没有副作用。

在某些场景下，很有可能因为这些默认值直造成成一些意想不到坑。

**案例 6：SNAPSHOT 包提供的新接口 "丢了"**

IDEA 默认不会更新 SNAPSHOT 包，第一次拉取后缓存该 Jar 包。

如果你在 IDEA 中设置 总是更新 SNAPSHOT 可能会发现，二方人员新开发的接口突然找不到了。

因此《手册》中有下面的规定：

> 【强制】线上应用不要依赖 SNAPSHOT 版本 (安全包除外)。
> 说明：不依赖 SNAPSHOT 版本是保证应用发布的幂等性。另外，也可以加快编译时的打包构建。



##### 2.2 环境相关的坑

**案例 1：配置错误**

很多时候由于我们粗心很有可能上线之前漏了一些配置导致出现一些线上问题。

比如实际开发中就出现某些人将 host 的值配置为 host+port 的形式，而该功能不影响核心流程，程序正常启动，但该功能没有正常工作。

**案例 2：测试时发现非常费解的问题**

开发中可能偶尔遇到非常诡异的问题，怎么看都应该是对的，但是就是不对，可能卡住很久。

此时要注意是不是环境搞错了。

因为很多大公司都会有好几套环境，如开发环境、QA 环境、QA 性能测试环境、预发环境、预发性能测试环境、和线上环境等。设置同样是 QA 环境还包括通用的 QA 环境和项目特有的 QA 环境。

遇到无法理解的问题，比如发布了项目没报错，但是就是访问不到，大概率是环境搞错了，要注意核对环境。

**案例 3：浏览器问题**

发现自己的浏览器访问总是有问题，而别人的浏览器可以，orz…

![图片描述](img/5dfc5ec40001ba6212100830.png)

图片来自谷歌浏览器官网。

此时可以尝试开无痕模式，大概率可以解决此类问题。



##### 2.3 测试中遇到的坑

**案例 1：场景没回归到**

比如你的服务接口提供给 Web 端、 小程序端 和 安卓 APP 端调用，修改了接口可能只自测了其中两个场景，然后代码上线发现唯一没覆盖的 小程序端有 BUG，囧…

【建议】自己的接口修改影响到的地方都要进行回归。

**案例 2: 只测试了正常值**

作为 Java 开发工程师我们测试自己接口时，总是习惯性测试正常值，而不会甚至想不到去测试异常数据。

接口上线以后，如果出现异常值，很可能就会导致线上问题。

【建议】做好参数防御，编码时要考虑异常情况。

**案例 3：服务启动调用了接口没走到断点**

在 QA 环境测试时可能会发现服务启动正常，调用的姿势也正确，在接口代码第一行断点都进不来，囧…

很多朋友可能会因为类似问题卡很久，其实此种情况很可能被拦截器拦截掉了…



##### 2.4 经验不足的坑

**案例 1：switch 缺少 default 语句**

《手册》中规定：在一个 switch 块内，都必须包含一个 default 语句并且放在最后，即使它什么代码也没有。

写 switch 的有些场景是根据类型来做不同的事情，比如商品类型，但是类型可能会增加，但是代码可能会滞后。

因此尽量要编写 default 语句，写 default 语句时要思考未来新增类型时会有什么影响？

【建议】如果走到 default 逻辑，如果可能会有不良影响的情况，即使不抛异常也要打印警告或者错误日志。

**案例 2： List 转 Map key 重复问题**

如果编写一个 List 转 Map 的函数，可能有些新手会这么写：

```java
private static Map<String, Integer> getResult(List<SimpleObject> data) {
    return data.stream()
            .collect(Collectors.toMap(SimpleObject::getName, SimpleObject::getValue));
}
```

这么写会存在一些问题吗，比如空指针异常，因为集合对象可能为 null, 集合的元素也可能为 null。

还有一个问题是 key 有可能重复，抛出：java.lang.IllegalStateException: Duplicate key 异常信息。

肿么办，orz…

可以参考如下写法：

```java
private static Map<String, Integer> getResult(List<SimpleObject> data) {
    if (CollectionUtils.isEmpty(data)) {
        return new HashMap<>(0);
    }
    return data.stream()
            .filter(Objects::nonNull)
            .filter(obj -> Objects.nonNull(obj.getValue()))
            .collect(Collectors.toMap(SimpleObject::getName, SimpleObject::getValue,
                    Integer::sum));
}
```

**案例 3：属性拷贝的坑**

开发时，很多人对自己要求不高，为了省事，大量使用 Bean 拷贝工具。

由于某些工具在特殊场景下会触发其自身的 BUG；还有多次属性拷贝通过代码很难看出哪些属性会填入了值，不必要地消耗了精力；修改对象属性时或者类型不对应时无法在编译器报错，增加了出错的风险；

具体建议参考 Java 属性拷贝的正确姿势小节。

**案例 4：越权问题**

比如你提供一个下载文档的接口，没有校验文档和当前用户的权限关系，而是直接根据文档 ID 就可以下载，那么如果用户发现规律，递增或者递减传入其他用户的文档 ID，就会发生安全问题。

【建议】不要只实现功能，对于需要隔离的资料，要重视权限的校验。

**案例 5：消息顺序性**

比如有这样一个场景，在短时间内发两次消息通知下游反查你的服务数据，然后更新到他们的库中。

两个消息可能被对方集群中的两台服务器分别消费，反查到的新数据可能先插到了对方的表中，而旧的数据更新到对方库中，导致旧值覆盖新值。

如果可以将两次消息合并为一次，这样就可以避免这个问题；如果是旧版本可丢弃的场景，也可以发送消息时带上时间戳或者版本号，下游更新数据时比对时间戳或版本号，时间更晚或者版本更旧直接丢弃。

**案例 6：暴露的服务修改签名**

提供给二方用的服务接口，由于需要优化或者升级，而且认为并没有人用，或者已经通知了使用方一起修改，于是大胆地修改了函数签名。

结果新的接口上线以后，发现未评估到的另外一个使用方调用报错，造成故障，orz…

【建议】暴露的接口不到万不得已不要修改函数签名，如果需要修改尽量提供新的接口。

**案例 7：没有兜底方案**

比如底层新增了一个必传参数，而这个参数是允许传默认的值，开发者认为前端肯定会传入的，就直接透传给底层，结果由于没覆盖所有场景，上线后有一处调用前端没传报错。

【建议】在条件允许的情况下尽可能设计兜底方案。

比如调用二方查询接口时 ip 是必传参数，我们开发时要求前端传给我们，但是我们实际开发时如果不传可以给个默认值，但是走到默认值逻辑时，我们要打印警告日志，引起我们的注意。

这样即使由于某些场景没覆盖到，功能还是 OK 的。



#### 3. 总结

本小节介绍了开发中可能遇到的典型坑点，并给出了对应的建议。

当然还有有各种各样的趟坑姿势，篇幅有限不可能一一介绍，也欢迎大家留言补充。

希望大家能够主动了解开发中可能遇到的坑点，主动积累，从别人的错误中吸取经验，少踩坑。

下一节将根据这几节讲到坑给出 Java 避坑宝典。



### Java避坑宝典



#### 1.前言

本章前面两节讲述了 Git 、数据库以及其他方面的各种坑点，并给出了针对某些具体坑点的建议。

本节将综合各种坑点和实际的开发经验，给出更宏观的避坑原则和具体的避坑经验[1](https://www.imooc.com/read/55/article/1174#fn1)。



#### 2.关于趟坑的一切



##### 2.1 趟坑的原因分析



###### 2.1.1 技术原理理解不到位

由于很多人不重视原理的学习，对很多技术理解不够深入，容易出现误用的情况。

比如 Git 理解不到位，本该从 master 拉取新分支开发新功能，却从另外一个项目分支拉取新的开发分支，导致项目上线带上了本不该发布的功能代码，从而出现故障。

比如对线程池的理解不够深入，使用 Executors 构造无界线程和无界队列的线程池，导致 OOM。



###### 2.1.2 没有养成好的习惯

开发前没有仔细梳理需求的习惯；开发时没有自我代码审查的习惯；测试时没有看错误日志的习惯；上线前没有检查各种配置的习惯；上线过程中和上线后没有关注线上日志的习惯。



###### 2.1.3 经验不够丰富

比如测试时没有充分评估影响面，导致漏测出现故障；

比如线上执行删除操作时，没有开启事务，也没有仔细反复检查；

比如都不认为自己依赖的服务会挂掉，不认为网络抖动会影响到自己的项目；

打印了敏感信息的日志，如果客户的住址、手机号、敏感账户等信息。

比如评估任务之前非常想当然，按照经验评估，而不是核实好每个接口，做好技术方案，留出灵活的时间，导致中间发现很多东西和自己想的不一样，项目经常延期。



###### 2.1.4 缺乏思考

很多人趟坑的另外一个重要原因是思考，不去思考为什么要这么设计，并且设计方案没有在头脑中推演。

由于缺乏思考，产品说啥就做啥，如果产品早期设计有问题，后期大量修改就会跟着被动修改。



###### 2.1.5 缺乏技术追求

很多人开发抱着一种完成任务就好的态度，缺乏技术追求。



##### 2.2 宏观避坑经验



###### 2.2.1 加强学习

重视官方文档。官方文档才是最权威最准确的参考资料，希望可以引起大家足够的重视；

加强专业知识的学习，可以重温大学最重要的四本课：操作系统、计算机网络、数据结构和算法、计算机组成原理，还可以购买黑皮书系列；

平时看书尽量购买经典的技术图书，有些技术图书要反复看；

平时开发时，如果不是特别忙可以进入到源码中，看源码的核心函数的注释和逻辑学习源码；

平时调试问题时，可以使用各种调试技巧来学习源码，比如查看调用栈、执行表达式等；

购买某个技术的专栏。专栏的质量一般都会偏高，大家可以通过专栏的学习，加深对某项技术的理解。



###### 2.2.2 养成好的习惯

要明确每个阶段自己的主要任务，甚至前一个阶段如果提前完成或者有时间可以提前准备后一个阶段的内容。

**编码前**：认真阅读需求文档，通过思维导图 和各种 UML 图来梳理需求和设计技术方案。

**开发中**多将自己的代码和 master 进行比对，及时发现自己代码中存在的问题。养成编写单元测试的习惯，单元测试

可以帮助你发现一些粗心导致的错误，帮你发现一些逻辑问题等。写代码时要注意编码规范，日志规范，注释规范等。

养成用静态代码检查插件检查自己代码的习惯，尽可能在编码阶段发现和解决问题。开发过程中一定本地编译通过再提交代码。

**测试阶段**一定重视错误日志，遇到问题及时修改；测试时不仅要测试正常情况，要以破坏者的角度自测。测试阶段要任何和预期不符的现象都要引起足够的重视。

**发布前**重新检查上线清单，核查配置是否都已经设置好，通知合作方就绪。

**发布阶段**及时观察错误日志、观察请求错误情况。



###### 2.2.3 积累开发经验

很多人的思维很奇怪，总是轻视别人的经验，这样容易走弯路，希望大家多从别人身上吸收经验。

每一个自己开发的大项目做完以后都要复盘，反思在从需求分析到上线整个过程中，本次项目自己存在哪些问题，编码中哪个设计不够完善，并总结出可供未来参考的经验。

了解微服务的核心构成，了解每种常用技术存在的主要问题和主要的解决方案。

多看《手册》、《Effective Java》、《重构》、《编写可维护代码的艺术》这饱含者前辈智慧的计算机开发经验的经典图书。

学习和实践高级的调试技巧，帮助自己更快定位问题，更好地学习源码。

要养成“面向失败编程”的思维习惯，要考虑各种依赖方出现问题对自己造成的影响，如果有可能可以设计出兜底方案。

多了解和实践软件设计的基本原则：高内聚、低耦合；

开发中多体会设计模式的六大原则：单一职责、里氏替换、依赖倒置、接口隔离、迪米特法则、开闭原则；学习经典设计模式的使用场景，优点、缺点和核心实现逻辑等。

> 注意：这里列举的规则并不是让大家背诵，而是希望大家能实实在在地去理解它们。

当发现公司其他团队或者团队的其他成员的代码出 BUG 甚至发生故障时，要了解出现问题的原因，自己后面开发时要注意规避。

如果有时间可以看看其他同事的技术方案，思考他做方案是为什么会这么设计，这样设计有什么好处等。

多和一些志同道合的朋友交流不涉及具体公司具体代码和业务内容的通用的开发经验。

> 积累一些具体的开发经验，如：
>
> **加开关**
>
> 如果对某个功能真得没把握，公司提供灰度发布，可以按比例切流量到包含新功能服的务器。如果没提供，可以用 apollo 做功能开关，验证有问题及时关闭；
>
> 如果某个功能有时需要打开，有时需要关闭，设计时可以通过apollo来控制，避免打开或关闭时需要修改源码发布。
>
> **加默认值**
>
> 比如调用某个接口，其中IP 是必传字段，如果不传会报错，如果场景允许，可以在不传时给一个默认值，并加上警告日志，这样就可以避免线上问题。
>
> **加白名单**
>
> 有些数据要走特殊逻辑，可以通过apollo 加入白名单。
>
> **预留拓展字段**
>
> 有些功能注定会拓展，可以预留一个拓展字段。
>
> 等等。



###### 2.2.4 做技术的思考者

通过本专栏的一些解读，很多朋友会发现：知道是什么，为什么可能比怎么做更重要。

因为很多人之所以不知道该怎么做，恰恰是因为并没有真正理解清楚概念，并没有认真思考过为什么要这样。



##### 2.3 排错技巧



###### 2.3.1 先猜想后验证

大家在遇到坑需要排查问题时，一定要根据已有知识根据可能性列举出几种最大概率的原因，然后再分别验证。



###### 2.3.2 控制变量法

有时候我们分析问题时会发现可能受到多种因素影响，此时我们要尽量让一个因素成为变量，其他的因素成为常量。

这样通过控制其他因素可以有效验证当前的因素是否是错误的主要原因。



###### 2.3.3 缩小范围

通过 F12 看请求和响应信息来定位问题是前端还是后端。

比如 4xx 响应码大都是前端错误； 5xx 响应码大都是后端错误。



###### 2.3.4 换环境大法

比如发现了诡异的问题，我们可以换环境，比如浏览器开无痕模式、换浏览器、换个项目、让同事用他的电脑试试等等。



###### 2.3.5 代码审查大法

代码审查小节给出了非常详细地讲解。

出现问题时，可以重新检查自己的代码逻辑，检查代码是否有性能问题，分析代码可能出错的原因。



###### 2.3.6 善用工具

**日志分析大法**

遇到问题优先查看日志，平时要熟练掌握 tail、grep、less 等操作日志的指令。

**调试大法**

前面章节中有讲到 Java 代码调试的基本技巧和高级技巧。

大家通过调试来分析问题时，要灵活运用这些调试技巧。

**抓包工具**

可以使用 tcpdump 命令抓取网络包来分析问题。

比如分析某个消息是否正常发送出去，可以通过该命令抓取自己的 topic，还可以通过消息的控制台观察消费情况来核实。

**反编译和反汇编**

对于涉及到源码或者语法糖的问题，大家还可以通过反编译和反汇编来分析问题。

**配套监控网站**

比如排查消息队列的问题，可以看公司的消息队列对应 topic 和 channel 的相关信息，辅助我们排查问题。

**arthas线上问题诊断工具**

可以借助 arthas的强大功能来分析线上问题。这需要在没有遇到问题时，本地多次训练学习。



###### 2.3.7 搜索引擎

建议大家灵活运用搜索引擎，善于从网上寻找类似错误，找到启发。

多去StackOverFlow 中去搜索问题。



###### 2.3.8 寻求帮助

紧急的问题自己无法处理，及时寻求帮助。

另外平时遇到一些很难排查的诡异的问题，自己卡住很久可以放一段时间再研究，会发现突然有思路，也可以让同事帮一起研究，有时候自己苦思冥想想不明白的事情，别人很快就可以想出原因，给出更合理的建议等。

最后，建议大家平时没遇到问题的时候多了解别人的坑，本地写 DEMO演练，这样真正遇到问题时才不会那么方。



#### 3.总结

本小节分析了开发中遇到坑的原因，主要包括基础不扎实、没有养成好的习惯、缺乏开发经验，缺乏思考等。针对这些问题给出了具体的建议。

希望大家能够更多地从别人的坑中学到经验，增强专业素养，提高编码水平，多产出少犯错。