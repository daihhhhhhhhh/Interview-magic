# Java类加载器工作机制详解

## 类加载的时机

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）和卸载（Unloading）7个阶段。其中 验证、准备、解析3个部分统称为链接（Link），这7个阶段的发生顺序如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/1401055-4aa13e716b87eab3.png?imageMogr2/auto-orient/strip|imageView2/2/w/571/format/webp)

class_loading.png

## 类加载的过程

接下来我们详细说一下Java虚拟机中类加载的全过程，也就是 加载、验证、准备、解析、初始化这5个阶段所执行的具体动作。

### 1、加载

"加载"是"类加载"（Class Loading）过程的一个阶段。在加载阶段，虚拟机需要完成以下3件事：

1. 通过一个的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转换为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。

### 2、验证

验证是 链接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

验证阶段大致上会完成下面4个阶段的校验动作：文件格式验证、元数据验证、字节码验证、符号引用验证。

### 3、准备

准备阶段是正式为类变量分配内存并设置变量初始值的阶段，这些变量所使用的内存都将在方法区中的进行分配。

这个阶段中有两个容易产生混淆的地方：
 首先，这时候进行内存分配的仅包括类变量（被static修饰的变量），而不包括示例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。
 其次，这里所说的初始值 "通常情况"下是数据类型的默认值。

### 4、解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

### 5、初始化

类初始化是类加载过程中的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过 自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说字节码）。

## 什么是类加载器

虚拟机设计团队吧类加载阶段中的 "通过一个类的全限定名来获取描述此类的二进制字节流"这个动作放到虚拟机外部去实现，以便让应用程序自己觉得如何去获取所需要的类。实现这个动作的代码模块称为 "类加载器"。

顾名思义，类加载器（ClassLoader ）用来加载 Java 类到 Java 虚拟机中。一般来说，Java 虚拟机使用 Java 类的方式如下：Java 源程序（.java 文件）在经过 Java 编译器编译之后就被转换成 Java 字节代码（.class 文件）。类加载器负责读取 Java 字节代码，并转换成 java.lang.Class类的一个实例。每个这样的实例用来表示一个 Java 类。

## ClassLoader 类

> 

A class loader is an object that is responsible for loading classes. The class ClassLoader is an abstract class. Given the binary name of a class, a class loader should attempt to locate or generate data that constitutes a definition for the class. A typical strategy is to transform the name into a file name and then read a "class file" of that name from a file system.

### ClassLoader 中与加载类相关的方法

| 方法                                                 | 说明                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| getParent()                                          | 返回该类加载器的父类加载器。                                 |
| loadClass(String name)                               | 加载名称为 name的类，返回的结果是 java.lang.Class类的实例。  |
| findClass(String name)                               | 查找名称为 name的类，返回的结果是 java.lang.Class类的实例。  |
| findLoadedClass(String name)                         | 查找名称为 name的已经被加载过的类，返回的结果是 java.lang.Class类的实例。 |
| defineClass(String name, byte[] b, int off, int len) | 把字节数组 b中的内容转换成 Java 类，返回的结果是 java.lang.Class类的实例。这个方法被声明为 final的。 |
| resolveClass(Class<?> c)                             | 链接指定的 Java 类。                                         |

loadClass(String name)方法源码如下：



```java
public abstract class ClassLoader {
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
}
```

loadClass方法主要流程：

1. 调用findLoadedClass(String) 方法检查这个类是否被加载过，如果已加载则跳到第4步，
2. 如果父加载器存在则调用父加载器调用 loadClass(String) 方法，否则，交给BootStrap ClassLoader 来加载此类，
3. 如果执行完上述2步骤 还没有找到对应的类，则调用当前类加载器的findClass(String) 。
4. 按照以上的步骤成功的找到对应的类，并且该方法接收的 resolve 参数的值为 true,那么就调用resolveClass(Class) 方法来处理类。

类加载器在成功加载某个类之后，会把得到的 java.lang.Class类的实例缓存起来。下次再请求加载该类的时候，类加载器会直接使用缓存的类的实例，而不会尝试再次加载。也就是说，对于一个类加载器实例来说，相同全名的类只加载一次，即 loadClass方法不会被重复调用。

java.lang.ClassLoader类的方法 loadClass()封装了类加载器的双亲委托模型的实现。因此，为了保证类加载器都正确实现 类加载器的双亲委托模型，在开发自己的类加载器时，最好不要覆写 loadClass()方法，而是覆写 findClass()方法。

### ClassLoader 体系

Java 中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由 Java 应用开发人员编写的。系统提供的类加载器主要有下面三个：

- BootStrap ClassLoader：称为启动类加载器，是Java类加载层次中最顶层的类加载器，负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等，可通过如下程序获得该类加载器从哪些地方加载了相关的jar或class文件：



```csharp
public class BootStrapTest
{
    public static void main(String[] args)
    {
      URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
      for (int i = 0; i < urls.length; i++) {
          System.out.println(urls[i].toExternalForm());
       }
    }
}
```

- Extension ClassLoader：称为扩展类加载器，负责加载Java的扩展类库，Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。默认加载JAVA_HOME/jre/lib/ext/目下的所有jar。
- App ClassLoader：称为系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件。一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。

除了系统提供的类加载器以外，开发人员可以通过继承java.lang.ClassLoader类的方式实现自己的类加载器，以满足一些特殊的需求。



```ruby
         Bootstrap ClassLoader
                  |
         Extension ClassLoader
                  |
            App ClassLoader
              /           \
    自定义ClassLoader1   自定义ClassLoader2
```

除了引导类加载器之外，所有的类加载器都有一个父类加载器。 给出的 getParent()方法可以得到。
 对于系统提供的类加载器来说，系统类加载器的父类加载器是扩展类加载器，而扩展类加载器的父类加载器是引导类加载器；对于开发人员编写的类加载器来说，其父类加载器是加载此类加载器 Java 类的类加载器。

## 类加载器的代理模式

类加载器在尝试自己去查找某个类的字节代码并定义它时，会先代理给其父类加载器，由父类加载器先去尝试加载这个类，依次类推。

当一个ClassLoader实例需要加载某个类时，它会试图亲自搜索某个类之前，先把这个任务委托给它的父类加载器，这个过程是由上至下依次检查的，首先由最顶层的类加载器Bootstrap ClassLoader试图加载，如果没加载到，则把任务转交给Extension ClassLoader试图加载，如果也没加载到，则转交给App ClassLoader进行加载，如果它也没有加载得到的话，则返回给委托的发起者，由它到指定的文件系统或网络等URL中加载该类。如果它们都没有加载到这个类时，则抛出ClassNotFoundException异常。

这种机制又称作做 类加载器的双亲委托模型。

### Java 虚拟机是如何判定两个 Class 是相同的

- JVM在判定两个class是否相同时，不仅要判断两个类名是否相同，而且要判断是否由同一个类加载器实例加载的。只有两者同时满足的情况下，JVM才认为这两个class是相同的。*

即便是同样的字节代码，被不同的类加载器加载之后所得到的类，也是不同的。比如一个 Java 类 com.example.Sample，编译之后生成了字节代码文件 Sample.class。两个不同的类加载器 ClassLoaderA和 ClassLoaderB
 分别读取了这个 Sample.class文件，并定义出两个 java.lang.Class类的实例来表示这个类。这两个实例是不相同的。对于 Java 虚拟机来说，它们是不同的类。试图对这两个类的对象进行相互赋值，会抛出运行时异常`java.lang.ClassCastException`。
 下面通过示例来具体说明。
 Java 类 com.example.Sample：



```java
 package com.example; 

 public class Sample { 
    private Sample instance; 

    public void setSample(Object instance) { 
        this.instance = (Sample) instance; 
    } 
 }
```

ClassIdentityDemo



```dart
public class ClassIdentityDemo {

    public static void main(String[] args) {

        new ClassIdentityDemo().testClassIdentity();
    }

    public void testClassIdentity() {
        String classDataRootPath = "F:\\github\\daily-codelab\\classloader-sample\\target\\classes";
        FileSystemClassLoader fscl1 = new FileSystemClassLoader(classDataRootPath);
        FileSystemClassLoader fscl2 = new FileSystemClassLoader(classDataRootPath);
        String className = "com.bytebeats.classloader.sample.ch3.Sample";
        try {
            Class<?> class1 = fscl1.loadClass(className);
            Object obj1 = class1.newInstance();
            Class<?> class2 = fscl2.loadClass(className);
            Object obj2 = class2.newInstance();
            Method setSampleMethod = class1.getMethod("setSample", java.lang.Object.class);
            setSampleMethod.invoke(obj1, obj2);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

FileSystemClassLoader



```java
import java.io.*;

/**
 * ${DESCRIPTION}
 *
 * @author Ricky Fung
 * @create 2017-03-12 17:24
 */
public class FileSystemClassLoader extends ClassLoader {
    private String rootDir;

    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = getClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] getClassData(String className) {
        String path = classNameToPath(className);
        try {
            InputStream in = new FileInputStream(path);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesNumRead = 0;
            while ((bytesNumRead = in.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesNumRead);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private String classNameToPath(String className) {
        return rootDir + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
    }

}
```

### 为什么要使用代理模式

了解了这一点之后，就可以理解代理模式的设计动机了。代理模式是为了保证 Java 核心库的类型安全。所有 Java 应用都至少需要引用 java.lang.Object类，也就是说在运行的时候，java.lang.Object这个类需要被加载到 Java 虚拟机中。如果这个加载过程由 Java 应用自己的类加载器来完成的话，很可能就存在多个版本的 java.lang.Object类，而且这些类之间是不兼容的。通过代理模式，对于 Java 核心库的类的加载工作由引导类加载器来统一完成，保证了 Java 应用所使用的都是同一个版本的 Java 核心库的类，是互相兼容的。

不同的类加载器为相同名称的类创建了额外的名称空间。相同名称的类可以并存在 Java 虚拟机中，只需要用不同的类加载器来加载它们即可。不同类加载器加载的类之间是不兼容的，这就相当于在 Java 虚拟机内部创建了一个个相互隔离的 Java 类空间。这种技术在许多框架中都被用到，例如OSGI、Web 容器(Tomcat)。

## 类加载器与 Web 容器

对于运行在 Java EE™容器中的 Web 应用来说，类加载器的实现方式与一般的 Java 应用有所不同。不同的 Web 容器的实现方式也会有所不同。以 Apache Tomcat 来说，每个 Web 应用都有一个对应的类加载器实例。该类加载器也使用代理模式，所不同的是它是首先尝试去加载某个类，如果找不到再代理给父类加载器。这与一般类加载器的顺序是相反的。这是 Java Servlet 规范中的推荐做法，其目的是使得 Web 应用自己的类的优先级高于 Web 容器提供的类。这种代理模式的一个例外是：Java 核心库的类是不在查找范围之内的。这也是为了保证 Java 核心库的类型安全。

## 源代码

所有源码均已上传给[Github](https://link.jianshu.com?t=https://github.com/TiFG/daily-codelab/tree/master/classloader-sample)

