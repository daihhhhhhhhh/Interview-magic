# spring事务,非事务方法与事务方法执行相互调用

写这篇文章的初衷呢就是最近遇到了一个spring事务的大坑.与其说是坑,还不如说是自己事务这块儿太薄弱导致的(自嘲下).

项目环境 sprinigboot

下面开始问题描述,发生的过程有点长,想直接看方案的直接跳过哦~; 

最近在做项目中有个业务是每天定时更新xx的数据,某条记录更新中数据出错,不影响整体数据,只需记录下来并回滚当条记录所关联的表数据; 好啊,这个简单,接到任务后,楼主我三下五除二就写完了,由于这个业务还是有些麻烦,我就在一个service里拆成了两个方法去执行,一个方法(A)是查询数据与验证组装数据,另外一个方法(B)更新这条数据所对应的表(执行的时候是方法A中调用方法B);由于这个数据是循环更新,所以我想的是,一条数据更新失败直接回滚此条数据就是,不会影响其他数据,其他的照常更新,所以我就在方法B上加了事务,方法A没有加; 以为很完美,自测一下正常,ok通过,再测试一下报错情况,是否回滚,一测,没回滚,懵圈儿?.以为代码写错了,改了几处地方,再测了几次,均没回滚.这下是真难受了.

好啦,写到这里,相信各位看官心里肯定在嘲讽老弟了,spring的传播机制都没搞明白(/难受); 

下面开始一步步分析解决问题:

首先我们来看下spring事务的传播机制及原因分析;

PROPAGATION_REQUIRED -- 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。
PROPAGATION_SUPPORTS -- 支持当前事务，如果当前没有事务，就以非事务方式执行。
PROPAGATION_MANDATORY -- 支持当前事务，如果当前没有事务，就抛出异常。
PROPAGATION_REQUIRES_NEW -- 新建事务，如果当前存在事务，把当前事务挂起。
PROPAGATION_NOT_SUPPORTED -- 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
PROPAGATION_NEVER -- 以非事务方式执行，如果当前存在事务，则抛出异常。
PROPAGATION_NESTED -- 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。 
spring默认的是PROPAGATION_REQUIRED机制,如果方法A标注了注解@Transactional 是完全没问题的,执行的时候传播给方法B,因为方法A开启了事务,线程内的connection的属性autoCommit=false,并且执行到方法B时,事务传播依然是生效的,得到的还是方法A的connection,autoCommit还是为false,所以事务生效;反之,如果方法A没有注解@Transactional 时,是不受事务管理的,autoCommit=true,那么传播给方法B的也为true,执行完自动提交,即使B标注了@Transactional ;

在一个Service内部，事务方法之间的嵌套调用，普通方法和事务方法之间的嵌套调用，都不会开启新的事务.是因为spring采用动态代理机制来实现事务控制，而动态代理最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了！

所以以上就是为什么我在没有标注事务注解的方法A里去调用标注有事务注解的方法B而没有事务滚回的原因;

 

看到这里,有的看官可能在想,你在方法A上标个注解不就完了吗?为什么非要标注在方法B上?

由于我这里是循环更新数据,调用一次方法B就更新一次数据,涉及到几张表,需要执行几条update sql, 一条数据更新失败不影响所有数据,所以说一条数据更新执行完毕后就提交一次事务,如果标注在方法A上,要所有的都执行完毕了才提交事务,这样子是有问题滴.

下边先上下代码:

方法A:无事务控制

![](https://img-blog.csdnimg.cn/20181117182421347.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM4MDI3NjU2,size_16,color_FFFFFF,t_70)

方法B:有事务控制

![](https://img-blog.csdnimg.cn/20181117183332207.png)

方法B处理失败手动抛出异常触发回滚:

![](https://img-blog.csdnimg.cn/2018111718315672.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM4MDI3NjU2,size_16,color_FFFFFF,t_70)

方法A调用方法B:

![](https://img-blog.csdnimg.cn/20181117182616543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM4MDI3NjU2,size_16,color_FFFFFF,t_70)

从上图可以看到,如果方法B中User更新出错后需要回滚RedPacket数据,所以User更新失败就抛出了继承自RuntimeException的自定义异常,并且在调用方把这个异常catch到重新抛出,触发事务回滚,但是并没有执行;

下面是解决方案:

   1.把方法B抽离到另外一个XXService中去,并且在这个Service中注入XXService,使用XXService调用方法B;

      显然,这种方式一点也不优雅,且要产生很多冗余文件,看起来很烦,实际开发中也几乎没人这么做吧?.反正我不建议采用此方案;

   2.通过在方法内部获得当前类代理对象的方式,通过代理对象调用方法B

   上面说了:动态代理最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了！

    所以我们就使用代理对象来调用,就会触发事务;

综上解决方案,我觉得第二种方式简直方便到炸. 那怎么获取代理对象呢? 这里提供两种方式:

   1.使用 ApplicationContext 上下文对象获取该对象;

   2.使用 AopContext.currentProxy() 获取代理对象,但是需要配置exposeProxy=true

我这里使用的是第二种解决方案,具体操作如下:

springboot启动类加上注解:@EnableAspectJAutoProxy(exposeProxy = true)

![](https://img-blog.csdnimg.cn/20181117185337876.png)

方法内部获取代理对象调用方法

![](https://img-blog.csdnimg.cn/20181117185508123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM4MDI3NjU2,size_16,color_FFFFFF,t_70)

完了后再测试,数据顺利回滚,至此,问题得到解决!

