# JAVA CPU占用过高问题排查(linux)

最近发现有一个服务在服务器上无响应，到服务器上一看，好家伙，java进程CPU一直100%以上

![img](https://img-blog.csdn.net/20160612234256674?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)





简单记录下我对这个问题的跟踪

首先当然要看下具体是java中哪个线程一直在占用cpu时间哈(说明下，我的java进程号是 26178)

1.根据java进程ID进行CPU占用排查  ps -mp 26178 -o THREAD,tid,time | sort -rn | more  （sort -rn 以数值的方式进行逆序排列）



![img](https://img-blog.csdn.net/20160612234310587?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)





2.根据1中查找到的CPU最高的排序中的结果，找出几个占用cpu时间比较高的TID，比如这里的26217 26182 26183

将进程ID转换为16进制，printf "%x\n" 26217



3.再使用jstack命名查询是哪个线程

TID十进制-》十六进制

26217 -》 6669

26182 -》 6646

26183 -》 6647

拿到线程ID的16进制之后，就可以从jstack中查找具体是对应的线程

jstack 26178 |grep 6669 -A 30

![img](https://img-blog.csdn.net/20160612234315581?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



可以发现，几个占用大量cpu时间的线程都是GC相关。

4.再次确认gc信息，查看gc time等信息，jstat -gcutil 26178 1000 100

![img](https://img-blog.csdn.net/20160612234321009?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



可以看到s0、s1、eden、old、metaspace都已经爆了，并且FGC次数一直在增加，但是却没有回收到任何空间，导致FGC一直在跑，进入循环，应该是程序存在内存泄露咯。（gc有日志，后续有空再出一篇简单分析gc日志的blog）

5.jmap -histo:live 26178 | more  简单查看对象的大小数目

6.dump内存，使用工具分析内存镜像，jmap -dump:live,format=b,file=problem.bin 26178

7.使用MAT（Memory Analyzer tool）进行数据分析，注意，如果步骤6中dump出来文件过大，需要设置MAT配置文件（MemoryAnalyzer.ini）的xmx参数的大小。