# java主线程捕获子线程中的异常

正常情况下，如果不做特殊的处理，在主线程中是不能够捕获到子线程中的异常的。

```java
public class ThreadExceptionRunner implements Runnable{
    @Override
    public void run() {
        throw new RuntimeException("error !!!!");
    }
}
```

使用线程执行上面的任务

``````java
import com.sun.glass.ui.TouchInputSupport;
 
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
 
public class ThreadExceptionDemo {
    public static void main(String[] args) {
        try {
            Thread thread = new Thread(new ThreadExceptionRunner());
            thread.start();
        } catch (Exception e) {
            System.out.println("========");
            e.printStackTrace();
        } finally {
        }
        System.out.println(123);
    }
}
``````

执行结果如下：

![img](https://img-blog.csdn.net/20180625224448285?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbGQ0NmNhdA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

如果想要在主线程中捕获子线程的异常，我们需要使用ExecutorService，同时做一些修改。

如下：

```java

import com.sun.glass.ui.TouchInputSupport;
 
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
 
public class ThreadExceptionDemo {
    public static void main(String[] args) {
        try {
            Thread thread = new Thread(new ThreadExceptionRunner());
            thread.start();
        } catch (Exception e) {
            System.out.println("========");
            e.printStackTrace();
        } finally {
        }
        System.out.println(123);
        ExecutorService exec = Executors.newCachedThreadPool(new HandleThreadFactory());
        exec.execute(new ThreadExceptionRunner());
        exec.shutdown();
    }
}
 
class MyUncaughtExceptionHandle implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("caught " + e);
    }
}
 
class HandleThreadFactory implements ThreadFactory {
    @Override
    public Thread newThread(Runnable r) {
        System.out.println("create thread t");
        Thread t = new Thread(r);
        System.out.println("set uncaughtException for t");
        t.setUncaughtExceptionHandler(new MyUncaughtExceptionHandle());
        return t;
    }
 
}
```

