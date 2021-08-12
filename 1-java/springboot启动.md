每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序，该方法所在类需要使用@SpringBootApplication注解，以及@ImportResource注解，@SpringBootApplication包括三个注解，功能如下：

@EnableAutoConfiguration：SpringBoot根据应用所声明的依赖来对Spring框架进行自动配置

@SpringBootConfiguration(内部为@Configuration)：被标注的类等于在spring的XML配置文件中(applicationContext.xml)，装配所有bean事务，提供了一个spring的上下文环境

@ComponentScan：组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run方法里的Booter.class所在的包路径下文件，所以最好将该启动类放到根包路径下



1、Every springboot program has a main entry, that is, the main method, which calls springapplication. Run() to start the whole springboot program. The class of this method needs to use the @SpringBootApplication annotation and the @ImportResource annotation.

@SpringBootApplication includes three annotations with the following functions: 

(1) @EnableAutoConfiguration: springboot automatically configures the spring framework according to the declared dependencies of the application
(2) @SpringBootConfiguration (internal @Configuration): the annotated class is equal to assembling all bean transactions in spring's XML configuration file (ApplicationContext. XML), providing a spring context environment
(3) @ComponentScan: component scan, which can automatically discover and assemble beans. By default, it scans the files in the package path of booter.class in the run method of springapplication, so it is better to put the startup class in the root package path

创建SpringApplication对象 ，然后调用run()方法。

第一步：可以看到，一开始是一个StopWatch类，该类的作用比较单一，就是记录springboot的启动用时，我们启动springboot完成后会在控制台springboot启动所花的时间，就是由它来完成的

第二步：创建SpringApplicationRunListeners

第三步：启动SpringApplicationRunListeners中所有的监听器

第四步：环境准备

第五步：打印Banner

第六步：创建上下文

