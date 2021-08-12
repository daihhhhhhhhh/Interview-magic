# JVM问题排查

本文将介绍JDK自带的JVM排查工具。其提供的排查工具有：

（1）jps：JVM Process Status Tool，显示系统内所有的JVM进程；

（2）jstat：JVM Statistics Monitoring Tool，可以收集JVM相关的运行数据；

（3）jinfo：Configuration Info for Java，显示JVM配置信息；

（4）jmap：Memory Map for Java，用于生成JVM的内存快照；

（5）jhat：JVM Heap Dump Browser，用于分析heapdump文件，它可以建立一个http/html服务，使用者可以在浏览器上查看分析结果；

（6）jstack：Stack Trace for Java，显示JVM的线程快照。

（7）jconsole:一个java GUI监视工具，可以以图表化的形式显示各种数据。并可通过远程连接监视远程的服务器VM。

## 一、jps

列出正在运行的虚拟机进程，并显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一id（Local Virtual Machine Identifier，简称LVMID）。相比其他命令来说，它的功能其实比较单一，但是它的使用频率确实非常高的，因为其他的JDK命令大多都需要输入虚拟机进程id。

注：看到这里小伙伴们可能就会比较困惑了，因为在大多数使用JDK命令时，我们都用的是pid（操作系统进程id）啊，那pid和LVMID到底有什么区别呢？其实，很负责任的告诉你，对于本地虚拟机进程来说，它俩无差别，用ps命令也可以查询到，但是如果同时启动多个虚拟机进程无法根据进程名称定位时，jps命令就派上用场了，可以输出主类名称，通过主类名称来区分。

jps命令格式：


jps命令格式
jps主要提供以下选项：

-q
只输出LVMID，省略主类名称；

-m
输出虚拟机进程启动时传给主类函数的参数；

-l
输出主类的完成package名称或者jar包完整路径名；

-v
输出虚拟机启动时的JVM参数

接下来我们来看看它的实际使用：

测试代码：

测试代码
使用结果：

jps使用结果

## 二、jstat

用于监控虚拟机运行状态信息。它可以显示本地或者远程虚拟机进程的内存、垃圾收集、JIT编译等运行数据。

jstat命令格式：


jstat命令格式

jstat命令稍许有些复杂，它主要有以下参数：
option：选项，jstat主要提供以下选项：

-class
监视类的装载/卸载数量、总空间以及类装载所耗时间；

-gc
监视java heap情况，包括eden区和两个survivor区、old区、永久区等的容量，已用空间和GC时间等信息；

-gccapacity
监视内容与-gc基本是一致的，-gccapacity的输出包括heap各个区域使用到的最大最小空间；

gcutil
监视内容同样与-gc基本一致，-gcutil的输出主要是heap各个区域使用空间占总空间百分比；

gccause
与-gcutil功能一致，但是会额外输出导致上一次gc的原因；

gcnew
监视young区gc情况；

gcnewcapacity
监视内容与-gcnew基本相同，-gcnewcapacity的输出包括使用到的最大最小空间；

-gcold
监视old区gc情况；

-gcoldcapacity
监视内容与-gcold基本相同，-gcoldcapacity的输出包括使用到的最大最小空间；

-gcpermcapacity
输出永久代使用到的最大最小空间；

注：JDK 8废除了永久代，引入了Metaspace，这个命令在JDK 8的环境下就不能使用了，那要看元数据空间相关情况，使用-gcmetacapacity即可

-compiler
输出JIT编译器编译过的方法以及耗时等信息；
-printcompilation
输出以及被JIT编译的方法
vmid：虚拟机进程id，这时候小伙伴们肯定又要开始疑惑了，这个vmid与lvmid又有什么区别？其实对于本地虚拟机进程，它俩没任何区别，但是如果是远程虚拟机进程，它俩就有区别了，远程虚拟机进程vmid格式应该是这样：
[protocol:][//] lvmid [@hostname[:port]/servername]；

interval：查询时间间隔；

count：查询次数。

注：如果参数interval和count省略则代表只查询一次；如果count省略的话，就会一直查询。来个简单的例子：jstat -gcnewcapacity 41503 1000，表示输出进程41503的young区使用及gc情况，每1000ms输出一次。

接下来就部分option给出实例，同时分析下输出。

jstat使用

jstat -class <vmid>



jstat -class输出
Loaded：装载的类的数量；

Bytes：装载类所占用的字节数；

Unloaded：卸载类数量；

Bytes：卸载类所占用的字节数；

Time：装载和卸载类所花时间。

jstat -compiler <vmid>



jstat -compiler输出
Compiled：编译任务执行数量；

Failed：编译任务执行失败数量；

Invalid：编译任务执行失效数量；

Time：编译任务消耗时间；

FailedType：最后一个编译失败任务的类型；

FailedMethod：最后一个编译失败任务所在的类及方法。

jstat -gc <vmid>



jstat -gc输出
S0C：young区中的第一个survivor区的大小；

S1C：young区中的第二个survivor区的大小；

S0U：young区中的第一个survivor区目前已使用空间；

S1U：young区中的第二个survivor区目前已使用空间；

EC：young区中的eden区的大小；

EU：young区中的eden区目前已使用空间；

OC：old区的大小；

OU：old区目前已使用空间；

MC：元数据区大小；

MU：元数据区使用大小；

CCSC：压缩类空间大小；

CCSU：压缩类空间使用大小；

YGC：young gc次数；

YGCT：young gc消耗时间；

FGC：full gc次数；

FGCT：full gc消耗时间；

GCT：gc消耗时间。

jstat -gcutil <vmid>



jstat -gcutil输出
S0：young区中的第一个survivor区的使用比例；

S1：young区中的第二个survivor区的使用比例；

E：young区中的eden区的使用比例；

O：old区使用比例；

M：元数据区使用比例；

CCS：压缩类空间使用比例；

YGC：young gc次数；

YGCT：young gc消耗时间；

FGC：full gc次数；

FGCT：full gc消耗时间；

GCT：gc消耗时间。

jstat -gccapacity <vmid>



jstat -gccapacity输出
NGCMN：young区最小容量；

NGCMX：young区最大容量；

NGC：当前young区容量；

S0C：young区中的第一个survivor区的大小；

S1C：young区中的第二个survivor区的大小；

EC：young区中的eden区的大小；

OGCMN：old区最小容量；

OGCMX：old区最大容量；

OGC：当前old区大小；

OC：当前old区的大小；

MCMN：元数据区最小容量；

MCMX：元数据区最大容量；

MC：当前元数据区大小；

CCSMN：压缩类空间最小容量；

CCSMX：压缩类空间最大容量；

CCSC：当前压缩类空间大小；

YGC：young gc次数；

FGC：old gc次数。

jstat -gcnew <vmid>



jstat -gcnew输出
S0C：young区中的第一个survivor区的大小；

S1C：young区中的第二个survivor区的大小；

S0U：young区中的第一个survivor区目前已使用空间；

S1U：young区中的第二个survivor区目前已使用空间；

TT：对象在young区存活的次数；

MTT：对象在young区存活的最大次数；

DSS：期望survivor区大小；

EC：young区中的eden区的大小；

EU：young区中的eden区目前已使用空间；

YGC：young gc次数；

YGCT：young gc消耗时间。

jstat -gcold <vmid>



jstat -gcold输出
MC：元数据空间大小；

MU：元数据空间使用大小；

CCSC：压缩类空间大小；

CCSU：压缩类空间使用大小；

OC：old区大小；

OU：old区使用大小；

YGC：young gc次数；

FGC：full gc次数；

FGCT：full gc消耗时间；

GCT：gc消耗时间。

jstat -gcmetacapacity <vmid>



jstat -gcmetacapacity输出
MCMN：元数据区最小容量；

MCMX：元数据区最大容量；

MC：元数据区大小；

CCSMN：压缩类空间最小容量；

CCSMX：压缩类空间最大容量；

CCSC：压缩类空间大小；

YGC：young gc次数；

FGC：full gc次数；

FGCT：full gc消耗时间；

GCT：gc消耗时间。

jstat -printcompilation <vmid>



jstat -printcompilation输出
Compiled：编译方法数量；

Size：编译方法的字节码数量；

Type：编译方法的编译类型；

Method：方法名称。

针对young区和old区相关capacity命令在这里就不做详细分析了，有兴趣的小伙伴自行敲一敲命令运行下，至于输出的表格列含义在前几个命令详细介绍中基本上都包括在内了。

注：在使用jstat命令输出的容量的单位是字节。

## 三、jinfo

用于实时查看和调整虚拟机参数。

jinfo命令格式


jinfo命令格式

jinfo命令主要有以下参数：
option：选项，jinfo主要提供以下选项：

-flag <name>
输出指定JVM参数值；

-flag [+|-]<name>
启用或禁用指定JVM参数；

-flag <name>=<value>
设置指定JVM参数值；

-flags
输出所有JVM参数

-sysprops
输出Java系统属性；

<no option>
不指定选项则输出所有的虚拟机参数和Java系统属性

pid：需要查看或者调整虚拟机参数的进程id

接下来我们来看看它的实际使用：



jinfo使用结果

## 四、jmap

用于生成堆内存快照（heapdump或者dump文件）。当然，如果不想使用jmap命令，也可以使用JVM参数来生成：

-XX:+HeapDumpOnOutOfMemoryError，如果虚拟机在出现OutOfMemory异常后生成dump文件；

-XX:+HeapDumpOnCtrlBreak，使用ctrl + break键让虚拟机生成dump文件。

当然，还有一种更暴力的方式就是在linux系统下，kill -3也可以让虚拟机生成dump文件。

jmap命令格式


jmap命令格式

相比jstat命令，jmap命令明显就简单的多了，就两个参数：
option：选项，jmap主要提供以下选项：

-dump
生成Java堆内存快照，使用格式为：-dump:[live, ]format=b,file=<filename>，使用hprof二进制形式，输出jvm的heap内容到文件<filename>，live子参数是可选的，如果指定live选项，就只dump出存活的对象；

finalizerinfo
显示在F-Queue中等待Finalizer线程执行finalize方法的对象；

-heap
显示heap详细信息，比如使用哪种回收器、参数配置、分代状态等；

histo
显示每个class的实例数目，内存占用，类全名信息。VM的内部类名字开头会加上前缀“*”.，如果带上live子参数，则只统计活的对象数量；

-permstat
以ClassLoader为统计口径显示永久代内存状态，需要注意的是，JDK 8将该option替换成了-clstats；

-F
强制生成dump文件，当虚拟机进程对-dump选项没有响应时可以使用。

pid：需要生成dump文件的进程id

dump文件在这里我就不做演示了，给一个简单的使用吧：



jmap -histo输出

从输出的结果可以清晰的看出每一个class的实例数目以及内存占用情况。

## 五、jstack

用户生成虚拟机当前时刻的线程快照（threaddump/javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈集合，生成线程快照的目的也就是为了定位线程出现长时间卡顿的原因。

jstack命令格式


jstack命令格式

跟jmap命令一样，jstack命令也只有两个参数：
option：选项，jstack主要提供以下选项：

-F
当线程出现长时间卡顿的时候，强制输出线程堆栈；

-l
除堆栈外，显示关于锁的附加信息；

-m
如果调用JNI方法，可以显示C/C++的堆栈。

pid：需要生成threaddump的进程id。

简单给一个使用吧还是：



jstack -l输出

从输出结果可以清晰的看到线程堆栈以及锁相关信息。具体的怎么根据threaddump分析定位问题最近暂时没有遇到，等遇到了再出文详细介绍啦~

## 六、jhat

与jmap命令搭配使用，分析jmap生成的dump文件。jhat内置一个微型的HTTP/HTML服务器，生成dump文件的分析结果后，可以在浏览器中进行查看。其实这个命令在实际生产中使用比较少，为什么不用的原因我总结下来大概有两点：第一呢，线上生产环境怎么可能允许你在线上机器直接分析dump文件啊~~~第二就是jhat的分析功能相比其他工具简直是太简陋了。

它的使用也很简单啊，jhat <dump文件名称>，等分析完就打开浏览器访问相应的地址+端口就可以愉快的开始分析dump文件了。

## 七、jconsole

jconsole是一个用java写的GUI程序，用来监控VM，并可监控远程的VM，非常易用，而且功能非常强。使用方法：命令行里打 jconsole，选则进程就可以了。

Console中关于内存分区的说明。
Eden Space (heap)： 内存最初从这个线程池分配给大部分对象。 
Survivor Space (heap)：用于保存在eden space内存池中经过垃圾回收后没有被回收的对象。 
Tenured Generation (heap)：用于保持已经在 survivor space内存池中存在了一段时间的对象。 
Permanent Generation (non-heap): 保存虚拟机自己的静态(refective)数据，例如类（class）和方法（method）对象。Java虚拟机共享这些类数据。这个区域被分割为只读的和只写的， 
Code Cache (non-heap):HotSpot Java虚拟机包括一个用于编译和保存本地代码（native code）的内存，叫做“代码缓存区”（code cache）