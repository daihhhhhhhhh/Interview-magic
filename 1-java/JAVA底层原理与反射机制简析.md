# JAVA底层原理与反射机制简析

**1.java底层原理简析**　

　　往往，在现在开发过程中，有很多操作，虽然功能都能去实现，但是在Jvm的内存分配上，是大有不同的，很可能两个不同的实现方式，性能上也会有或多或少差异……

　　![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190505171340561-1661446705.png)

例如:

private Integer name = 4;

private static Integer name = 4;

private final static Integer name 4;

private final Integer name =4;

它们的name对应的值都是4,但是否思考过，究竟有何实际区别呢?

　　所以接下来，我们进入主题，去讲解究竟有什么区别吧!


**1.1 Java里的堆(heap)栈(stack)和方法区(method)**

**栈区:**

1.每个线程包含一个栈区，栈中只保存基础数据类型的对象和自定义对象的引用(不是对象)，对象都存放在堆区中

2.每个栈中的数据(原始类型和对象引用)都是私有的，其他栈不能访问。

3.栈分为3个部分：基本类型变量区、执行环境上下文、操作指令区(存放操作指令)。

 
**堆区:**
1.存储的全部是对象，每个对象都包含一个与之对应的class的信息。(class的目的是得到操作指令)

2.jvm只有一个堆区(heap)被所有线程共享，堆中不存放基本类型和对象引用，只存放对象本身 。

**方法区:**

1.又叫静态区，跟堆一样，被所有的线程共享。方法区包含所有的class和static变量。

2.方法区中包含的都是在整个程序中永远唯一的元素，如class，static变量。 

　　Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间。有的区域随着虚拟机进程的启动而存在，有些区域则是依赖用户线程的启动和结束而建立和销毁。
JVM内存模型可以分为两个部分，如下图所示，堆和方法区是所有线程共有的，而虚拟机栈，本地方法栈和程序计数器则是线程私有的。下面我们就来分析一下这些不同区域的作用。

**总结一下:**
　　堆栈(Stack)是操作系统在建立某个进程时或者线程(在支持多线程的操作系统中是线程)为这个线程建立的存储区域，该区域具有先进后出的特性。每一个Java应用都唯一对应一个JVM实例，每一个实例唯一对应一个堆。应用程序在运行中所创建的所有类实例或数组都放在这个堆中,并由应用所有的线程共享，Java中分配堆内存是自动初始化的。Java中所有对象的存储空间都是在堆中分配的，但是这个对象的引用却是在堆栈中分配,也就是说在建立一个对象时从两个地方都分配内存，在堆中分配的内存实际建立这个对象，而在堆栈中分配的内存只是一个指向这个堆对象的指针(引用)而已。
方法区存着类的信息，常量和静态变量，即类被编译后的数据。

**JVM内存模型图:**

**![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190505172116453-1797404337.png)**

 

 

上述貌似有点难懂，我接下来自己画图来解释一下!

 ![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190505172241607-1199233349.png)

 

 

 

**方法区剖析图:**

**![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190505172328963-658939920.png)**

 

 


讲了这么多，我们来点干货演示一下!
比较类里面的数值是否相等时，用equals()方法；当测试两个包装类的引用是否指向同一个对象时，用==，下面用例子说明上面的理论。

**示例一：**

String str1 = "abc";

String str2 = "abc";

System.out.println(str1 == str2); //true
可以看出str1和str2是指向同一个对象的。

**示例二：**
String str1 = new String("abc");

String str2 = new String("abc");

System.out.println(str1 == str2); // false
用new的方式是生成不同的对象。每一次生成一个。


**示例一和示例二解析:**
用第一种方式创建多个”abc”字符串,在内存中 其实只存在一个对象而已。而对于String str = new String("abc")；的代码，则一概在堆中创建新对象，而不管其字符串值是否相等，是否有必要创建新对象，从而加重了程序的负担。 

**示例三：**
String s0="kvill"; 

String s1="kvill";

String s2="kv" + "ill";

System.out.println( s0==s1 );

System.out.println( s0==s2 );

结果为：true true 

**示例四：**
String s0="kvill";

String s1=new String("kvill");

String s2="kv" + new String("ill");

System.out.println( s0==s1 );

System.out.println( s0==s2 );

System.out.println( s1==s2 );

结果为：false false false

 


**示例三和示例四解析:**
因为例子中的 s0和s1中的”kvill”都是字符串常量，它们在编译期就被确定了，所以s0==s1为true；而"kv”和"ill”也都是字符串常量，当一个字符串由多个字符串常量连接而成时，它自己肯定也是字符串常量，所以s2也同样在编译期就被解析为一个字符串常量，所以s2也是常量池中” kvill”的一个引用。所以我们得出s0==s1==s2；用new String() 创建的字符串不是常量，不能在编译期就确定，所以new String() 创建的字符串不放入常量池中，它们有自己的地址空间。

 

**小问题:**String s = new String(“xyz”);产生几个对象？

**答案:**1个或者2个。
**解析:**对于通过new产生一个字符串（假设为”xyz”）时，会先去常量池中查找是否已经有了”xyz”对象，如果没有则在常量池中创建一个此字符串对象，然后堆中再创建一个引用常量池中此”xyz”对象的拷贝对象。

**static final 和final修饰常量的区别:** 

static 强调只有一份，final 只是说明是一个常量，final定义的基本类型的值是不可改变的，但是final定义的引用对象的值是可以改变的。

 

不知道在说什么，来点干货!!

![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190506083523397-625965215.png)

```
import java.util.Random;

/**
 * 这个例子想说明一下static final 与 final的区别
 */
public class Test1 {
    //47作为随机种子，为的就是产生随机数。
    private static Random rand = new Random(47);
    private final int A = rand.nextInt(30);
    private static final int B = rand.nextInt(30);

    public static void main(String[] args) {
        Test1 test1 = new Test1();
        System.out.println("test1 : " + "A=" + test1.A); //5
        System.out.println("test1 : " + "B=" + test1.B); //8
        Test1 test2 = new Test1();
        System.out.println("test2 : " + "A=" + test2.A); //13
        System.out.println("test2 : " + "B=" + test2.B); //8
    }
}
```



 

最后讲一下关于引用变量的内存分配!也是我们最为核心关注的内容!

**引用类型：**是指除了基本的变量类型之外的所有类型（如通过 class 定义的类型）。所有的类型在内存中都会分配一定的存储空间(形参在使用的时候也会分配存储空间,方法调用完成之后,这块存储空间自动消失), 基本的变量类型只有一块存储空间(分配在栈中), 而引用类型有两块存储空间(一块在栈中,一块在堆中)。

接下来我们继续用模型图分析!

**引用数据类型内存分配图一：**

**![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190506084544619-240739287.png)**

 

 

**引用数据类型内存分配图二：**

![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190506084815786-1657361380.png)

 

可以看到引用数据类型内存分配图一和图二有所不同，其实图二只是对图一的更深层的剖析，图二这里存在一个句柄的概念。

 

**这里举两个具体实例去作分析**

**实例一:**

 **![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190506085251260-1459725758.png)**

 


**实例二:**

**![img](https://img2018.cnblogs.com/blog/1664502/201905/1664502-20190506085105820-1660306178.png)**

 


**分析：**通过上面的代码和图形示例不难看出，a 和 b 是不同的两个引用，我们使用了两个定义语句来定义它们。但它们的值是一样的，都指向同一个对象"This is a Text!"。但要注意String 对象的值本身
是不可更改的 (像 b = "World"; b = a; 这种情况不是改变了 "World" 这一对象的值，而是改变了它的引用 b 的值使之指向了另一个 String 对象 a)。
这里我描述了两个要点：
（1） 引用是一种数据类型（保存在stack中），保存了对象在内存（heap，堆空间）中的地址，这种类型即不是我们平时所说的简单数据类型也不是类实例(对象)；
（2）不同的引用可能指向同一个对象，换句话说，一个对象可以有多个引用，即该类类型的变量。

 


**2.java的反射机制简析**


**2.1 反射机制的概念:**
指在运行状态中,对于任意一个类,都能够知道这个类的所有属性和方法,对于任意一个对象,都能调用它的任意一个方法.这种动态获取信息,以及动态调用对象方法的功能叫java语言的反射机制.


**2.2 反射机制的应用:**
生成动态代理,面向切片编程(在调用方法的前后各加栈帧).


**2.3 反射机制的原理:**
1 首先明确的概念: 一切皆对象----类也是对象.
2 然后知道类中的内容 :modifier  constructor  field  method.
3 其次明白加载: 当Animal.class在硬盘中时,是一个文件,当载入到内存中,可以认为是一个对象,是java.lang.class的对象.


**2.4 反射机制简单实例代码：**

 

```
public class CustomService
{
     //登录
             public String login(String name,String pwd){
         if("admin".equals(name) && "123".equals(pwd)){
              return “success”;
         }
         return “fail”;
     }
     //退出
             public void logout(){
         System.out.println("系统已安全退出！");
     }
}
public class ReflectTest
{
     public static void main(String[] args) throws Exception{
         //1.获取类
         Class c = Class.forName("CustomerService");
         //获取某个特定的方法
         //通过：方法名+形参列表
         Method m = c.getDeclaredMethod("login",String.class,String.class);
         //通过反射机制执行login方法.
         Object obj = c.newInstance();
         //调用obj对象的m方法,传递"admin""123"参数，方法的执行结果是retValue
         Object retValue = m.invoke(obj, "admin","123");
         System.out.println(retValue); //success
     }
}
```



 