在JDK7以及其前期的JDK版本中，堆内存通常被分为三块区域Nursery内存(young generation)、长时内存(old generation)、永久内存(Permanent Generation for VM Matedata)，下图是Java7.

![img](https://upload-images.jianshu.io/upload_images/11464886-07b1ac1bdca5eb2a.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

其中最上一层是Nursery内存，一个对象被创建以后首先被放到Nursery中的Eden内存中，如果存活期超两个Survivor之后就会被转移到长时内存中(Old Generation)中

永久内存中存放着对象的方法、变量等元数据信息。通过如果永久内存不够，我们就会得到如下错误：

java.lang.OutOfMemoryError: PermGen

而在JDK8中情况发生了明显的变化，就是一般情况下你都不会得到这个错误，原因在于JDK8中把存放元数据中的永久内存从堆内存中移到了本地内存(native memory)中，JDK8中JVM堆内存结构就变成了如下：

![img](https://upload-images.jianshu.io/upload_images/11464886-4bf1eb2bde86ca69.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

这样永久内存就不再占用堆内存，它可以通过自动增长来避免JDK7以及前期版本中常见的永久内存错误(java.lang.OutOfMemoryError: PermGen)，也许这个就是你的JDK升级到JDK8的理由之一吧。当然JDK8也提供了一个新的设置Matespace内存大小的参数，通过这个参数可以设置Matespace内存大小，这样我们可以根据自己项目的实际情况，避免过度浪费本地内存，达到有效利用。

