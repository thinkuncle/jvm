# 类加载的过程

类加载过程包括 5 个阶段：加载、验证、准备、解析和初始化。

## 加载

### 加载的过程

“加载”是“类加载”过程的一个阶段，不能混淆这两个名词。在加载阶段，虚拟机需要完成 3 件事：

* 通过类的全限定名获取该类的二进制字节流。
* 将二进制字节流所代表的静态结构转化为方法区的运行时数据结构。
* 在内存中创建一个代表该类的 java.lang.Class 对象，作为方法区这个类的各种数据的访问入口。

### 获取二进制字节流

对于 Class 文件，虚拟机没有指明要从哪里获取、怎样获取。除了直接从编译好的 .class 文件中读取，还有以下几种方式：

* 从 zip 包中读取，如 jar、war等
* 从网络中获取，如 Applect
* 通过动态代理计数生成代理类的二进制字节流
* 由 JSP 文件生成对应的 Class 类
* 从数据库中读取，如 有些中间件服务器可以选择把程序安装到数据库中来完成程序代码在集群间的分发。

### “非数组类”与“数组类”加载比较

* 非数组类加载阶段可以使用系统提供的引导类加载器，也可以由用户自定义的类加载器完成，开发人员可以通过定义自己的类加载器控制字节流的获取方式（如重写一个类加载器的 loadClass\(\) 方法）
* 数组类本身不通过类加载器创建，它是由 Java 虚拟机直接创建的，再由类加载器创建数组中的元素类。

### 注意事项

* 虚拟机规范未规定 Class 对象的存储位置，对于 HotSpot 虚拟机而言，Class 对象比较特殊，它虽然是对象，但存放在方法区中。
* 加载阶段与连接阶段的部分内容交叉进行，加载阶段尚未完成，连接阶段可能已经开始了。但这两个阶段的开始实践仍然保持着固定的先后顺序。

## 验证

### 验证的重要性

验证阶段确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

### 验证的过程

* 文件格式验证  验证字节流是否符合 Class 文件格式的规范，并且能被当前版本的虚拟机处理，验证点如下： 
  * 是否以魔数 0XCAFEBABE 开头
  * 主次版本号是否在当前虚拟机处理范围内
  * 常量池是否有不被支持的常量类型
  * 指向常量的索引值是否指向了不存在的常量
  * CONSTANT\_Utf8\_info 型的常量是否有不符合 UTF8 编码的数据
  * ......
* 元数据验证  对字节码描述信息进行语义分析，确保其符合 Java 语法规范。
* 字节码验证  本阶段是验证过程中最复杂的一个阶段，是对方法体进行语义分析，保证方法在运行时不会出现危害虚拟机的事件。
* 符号引用验证 本阶段发生在解析阶段，确保解析正常执行。

## 准备

准备阶段是正式为类变量（或称“静态成员变量”）分配内存并设置初始值的阶段。这些变量（不包括实例变量）所使用的内存都在方法区中进行分配。

初始值“通常情况下”是数据类型的零值（0, null...），假设一个类变量的定义为：

```java
public static int value = 123;
```

那么变量 value 在准备阶段过后的初始值为 0 而不是 123，因为这时候尚未开始执行任何 Java 方法。

存在“特殊情况”：如果类字段的字段属性表中存在 ConstantValue 属性，那么在准备阶段 value 就会被初始化为 ConstantValue 属性所指定的值，假设上面类变量 value 的定义变为：

```java
public static final int value = 123;
```

那么在准备阶段虚拟机会根据 ConstantValue 的设置将 value 赋值为 123。

## 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

## 初始化

类初始化阶段是类加载过程的最后一步，是执行类构造器 &lt;clinit&gt;\(\) 方法的过程。

&lt;clinit&gt;\(\) 方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static {} 块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的。

静态语句块中只能访问定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块中可以赋值，但不能访问。如下方代码所示：

```java
public class Test {
    static {
        i = 0;  // 给变量赋值可以正常编译通过
        System.out.println(i);  // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

&lt;clinit&gt;\(\) 方法不需要显式调用父类构造器，虚拟机会保证在子类的 &lt;clinit&gt;\(\) 方法执行之前，父类的 &lt;clinit&gt;\(\) 方法已经执行完毕。

由于父类的 &lt;clinit&gt;\(\) 方法先执行，意味着父类中定义的静态语句块要优先于子类的变量赋值操作。如下方代码所示：

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
    System.out.println(Sub.B); // 输出 2
}
```

&lt;clinit&gt;\(\) 方法不是必需的，如果一个类没有静态语句块，也没有对类变量的赋值操作，那么编译器可以不为这个类生成 &lt;clinit&gt;\(\) 方法。

接口中不能使用静态代码块，但接口也需要通过 &lt;clinit&gt;\(\) 方法为接口中定义的静态成员变量显式初始化。但接口与类不同，接口的 &lt;clinit&gt;\(\) 方法不需要先执行父类的 &lt;clinit&gt;\(\) 方法，只有当父接口中定义的变量使用时，父接口才会初始化。

虚拟机会保证一个类的 &lt;clinit&gt;\(\) 方法在多线程环境中被正确加锁、同步。如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的 &lt;clinit&gt;\(\) 方法。

（完）
---
👉 [Previous](/08-load-class-time.md)
👉 [Next](/10-class-loader.md)
👉 [Back to README](../README.md)