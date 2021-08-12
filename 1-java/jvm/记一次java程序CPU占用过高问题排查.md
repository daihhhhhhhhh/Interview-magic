# 记一次java程序CPU占用过高问题排查

问题是这样的，将项目部署到服务器上后，发现应用程序的响应速度非常慢，于是开始进行了排查。

TOP
首先查看系统资源占用信息，TOP看一下

![](https://img-blog.csdn.net/20171129123034411?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

发现正在运行的JAVA项目CPU占用率很高，百分之200左右了，那么问题一定出在这个程序中

 

 

Ps -mp pid -o THREAD,tid,time


再通过ps命令查看这个程序的线程信息,tid代码线程ID，time代表这个线程的已运行时间

由上面TOP可知进程ID为15669

![](https://img-blog.csdn.net/20171129123645850?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](https://img-blog.csdn.net/20171129123656523?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

于是可以看到这个进程中有3个线程的CPU占用率很高，并且它们目前也运行了13分钟了，它们的TID分别为16068,16069,16071

 

进制转换，2HEX
再将这3个TID转为16进制，为等会在jstack中查找方便

 

 

Printf “%x\n” number

![](https://img-blog.csdn.net/20171129123954968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

 

得到这三个数的16进制为别为3ec4,3ec5,3ec7

 

jstack查看进程信息
有了线程ID的16进制后，再在jstack中查看进程堆栈信息(之所有拿到TID信息，主要是为了查找方便)

通过jstack -pid 再grep查询

 ![](https://img-blog.csdn.net/20171129124347613?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



从这里能看出，这3个线程目前还处于运行状态的

再通过jstack查看详细点的信息

![](https://img-blog.csdn.net/20171129124528670?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcHVoYWl5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
