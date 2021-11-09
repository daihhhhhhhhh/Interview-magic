# TCP通信过程中time_wait和close_wait产生过多的原因和解决方法

![](https://img-blog.csdnimg.cn/20200917180653986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDcxODc5NA==,size_16,color_FFFFFF,t_70#pic_center)

## time_wait过多产生原因

正常的TCP客户端连接在关闭后，会进入一个TIME_WAIT的状态，持续的时间一般在1-4分钟，对于连接数不高的场景，1-4分钟其实并不长，对系统也不会有什么影响，
  但如果短时间内（例如1s内）进行大量的短连接，则可能出现这样一种情况：客户端所在的操作系统的socket端口和文件描述符被用尽，系统无法再发起新的连接！

举例来说：
  假设每秒建立了1000个短连接（Web场景下是很常见的，例如每个请求都去访问memcached），假设TIME_WAIT的时间是1分钟，则1分钟内需要建立6W个短连接，由于TIME_WAIT时间是1分钟，这些短连接1分钟内都处于TIME_WAIT状态，都不会释放，而Linux默认的本地端口范围配置是：net.ipv4.ip_local_port_range = 32768 61000不到3W，因此这种情况下新的请求由于没有本地端口就不能建立了。

## time_wait过多解决方法

可以改为长连接，但代价较大，长连接太多会导致服务器性能问题；
修改ipv4.ip_local_port_range，增大可用端口范围，但只能缓解问题，不能根本解决问题；
客户端程序中设置socket的SO_LINGER选项；
客户端机器打开tcp_tw_recycle和tcp_timestamps选项；
客户端机器打开tcp_tw_reuse和tcp_timestamps选项；
客户端机器设置tcp_max_tw_buckets为一个很小的值；
参考：https://www.cnblogs.com/cheyunhua/p/9082674.html

## close_wait过多原因

  close_wait 按照正常操作的话应该很短暂的一个状态，接收到客户端的fin包并且回复客户端ack之后，会继续发送FIN包告知客户端关闭关闭连接，之后迁移到Last_ACK状态。但是close_wait过多只能说明没有迁移到Last_ACK，也就是服务端是否发送FIN包，只有发送FIN包才会发生迁移，所以问题定位在是否发送FIN包。FIN包的底层实现其实就是调用socket的close方法，这里的问题出在没有执行close方法。**说明服务端socket忙于读写**。

## 4.close_wait过多的解决方案

代码层面做到
第一：使用完socket调用close方法；
第二：socket读控制，当读取的长度为0时（读到结尾），立即close；
第三：如果read返回-1，出现错误，检查error返回码，有三种情况：INTR（被中断，可以继续读取），WOULDBLOCK（表示当前socket_fd文件描述符是非阻塞的，但是现在被阻塞了），AGAIN（表示现在没有数据稍后重新读取）。如果不是AGAIN，立即close
可以设置TCP的连接时长keep_alive_time还有tcp监控连接的频率以及连接没有活动多长时间被迫断开连接