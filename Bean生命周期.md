1、实例化一个Bean－－也就是我们常说的new；

2、按照Spring上下文对实例化的Bean进行配置－－也就是IOC注入；

3、如果这个Bean已经实现了BeanNameAware接口，会调用它实现的setBeanName(String)方法，此处传递的就是Spring配置文件中Bean的id值

4、如果这个Bean已经实现了BeanFactoryAware接口，会调用它实现的setBeanFactory(setBeanFactory(BeanFactory)传递的是Spring工厂自身（可以用这个方式来获取其它Bean，只需在Spring配置文件中配置一个普通的Bean就可以）；

5、如果这个Bean已经实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文（同样这个方式也可以实现步骤4的内容，但比4更好，因为ApplicationContext是BeanFactory的子接口，有更多的实现方法）；

6、如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessBeforeInitialization(Object obj, String s)方法，BeanPostProcessor经常被用作是Bean内容的更改，并且由于这个是在Bean初始化结束时调用那个的方法，也可以被应用于内存或缓存技术；

7、如果Bean在Spring配置文件中配置了init-method属性会自动调用其配置的初始化方法。

8、如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessAfterInitialization(Object obj, String s)方法、；

9、当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用那个其实现的destroy()方法；

10、最后，如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法。



![img](https://img2018.cnblogs.com/blog/1329146/201904/1329146-20190416211636679-1788097403.png)



1.spring 的生命周期粗粒度的可以分为4个阶段

   第一阶段：实例化（Instantiation）

           //实例化是指Bean 从Bean到Object  
           Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);

第二阶段： 属性赋值

 

 第三阶段：初始化(Initialization)

          初始化前： org.springFrameWork.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization
    
          初始化中         org.springFrameWork.bean.InitializingBean#afterPropertiesSet
    
          初始化后org.springFrameWork.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization

  


第四阶段：销毁

                     org.springFrameWork.bean.factory.DisposableBean#destory

备注：.spring的核心就是Bean,Bean的生命周期是通过spring-context（上下文）控制的，而spring-context又基于spring-core进行的,只有Bean进行初始化后被IOC容器所管理，我们才可以在我们的应用中调用任意已经初始化的Bean.