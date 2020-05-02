# Spring事务隔离级别

ISOLATION_DEFAULT：用底层数据库的默认隔离级别，数据库管理员设置什么就是什么
ISOLATION_READ_UNCOMMITTED（未提交读）：最低隔离级别、事务未提交前，就可被其他事务读取（会出现幻读、脏读、不可重复读）
ISOLATION_READ_COMMITTED（提交读）：一个事务提交后才能被其他事务读取到（该隔离级别禁止其他事务读取到未提交事务的数据、所以还是会造成幻读、不可重复读）、sql server默认级别
ISOLATION_REPEATABLE_READ（可重复读）：可重复读，保证多次读取同一个数据时，其值都和事务开始时候的内容是一致，禁止读取到别的事务未提交的数据（该隔离基本可防止脏读，不可重复读（重点在修改），但会出现幻读（重点在增加与删除））（MySql默认级别，更改可通过set transaction isolation level 级别）
ISOLATION_SERIALIZABLE（序列化）：代价最高最可靠的隔离级别（该隔离级别能防止脏读、不可重复读、幻读）
丢失更新：两个事务同时更新一行数据，最后一个事务的更新会覆盖掉第一个事务的更新，从而导致第一个事务更新的数据丢失，这是由于没有加锁造成的；
幻读：同样的事务操作过程中，不同时间段多次（不同事务）读取同一数据，读取到的内容不一致（一般是行数变多或变少）。
脏读：一个事务读取到另外一个未提及事务的内容，即为脏读。
不可重复读：同一事务中，多次读取内容不一致（一般行数不变，而内容变了）。

# Spring传播行为

PROPAGATION_REQUIRED：支持当前事务，如当前没有事务，则新建一个。
PROPAGATION_SUPPORTS：支持当前事务，如当前没有事务，则已非事务性执行（源码中提示有个注意点，看不太明白，留待后面考究）。
PROPAGATION_MANDATORY：支持当前事务，如当前没有事务，则抛出异常（强制一定要在一个已经存在的事务中执行，业务方法不可独自发起自己的事务）。
PROPAGATION_REQUIRES_NEW：始终新建一个事务，如当前原来有事务，则把原事务挂起。
PROPAGATION_NOT_SUPPORTED：不支持当前事务，始终已非事务性方式执行，如当前事务存在，挂起该事务。
PROPAGATION_NEVER：不支持当前事务；如果当前事务存在，则引发异常。
PROPAGATION_NESTED：如果当前事务存在，则在嵌套事务中执行，如果当前没有事务，则执行与 PROPAGATION_REQUIRED 类似的操作（注意：当应用到JDBC时，只适用JDBC 3.0以上驱动）。

# Spring实现事务管理

## **编程式事务：编码方式实现事务管理（代码演示为JDBC事务管理）**

### 1）PlatformTransactionManager事务管理器

```java

import java.util.Map;
import javax.annotation.Resource;
import javax.sql.DataSource;
import org.apache.log4j.Logger;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;
 
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath:spring-public.xml" })
public class test {
	@Resource
	private PlatformTransactionManager txManager;
	@Resource
	private  DataSource dataSource;
	private static JdbcTemplate jdbcTemplate;
	Logger logger=Logger.getLogger(test.class);
    private static final String INSERT_SQL = "insert into testtranstation(sd) values(?)";
    private static final String COUNT_SQL = "select count(*) from testtranstation";
	@Test
	public void testdelivery(){
		//定义事务隔离级别，传播行为，
	    DefaultTransactionDefinition def = new DefaultTransactionDefinition();  
	    def.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);  
	    def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);  
	    //事务状态类，通过PlatformTransactionManager的getTransaction方法根据事务定义获取；获取事务状态后，Spring根据传播行为来决定如何开启事务
	    TransactionStatus status = txManager.getTransaction(def);  
	    jdbcTemplate = new JdbcTemplate(dataSource);
	    int i = jdbcTemplate.queryForInt(COUNT_SQL);  
	    System.out.println("表中记录总数："+i);
	    try {  
	        jdbcTemplate.update(INSERT_SQL, "1");  
	        txManager.commit(status);  //提交status中绑定的事务
	    } catch (RuntimeException e) {  
	        txManager.rollback(status);  //回滚
	    }  
	    i = jdbcTemplate.queryForInt(COUNT_SQL);  
	    System.out.println("表中记录总数："+i);
	}
```

### 2）使用TransactionTemplate，该类继承了接口DefaultTransactionDefinition，用于简化事务管理，事务管理由模板类定义，主要是通过TransactionCallback回调接口或TransactionCallbackWithoutResult回调接口指定，通过调用模板类的参数类型为TransactionCallback或TransactionCallbackWithoutResult的execute方法来自动享受事务管理。

TransactionTemplate模板类使用的回调接口：

TransactionCallback：通过实现该接口的“T doInTransaction(TransactionStatus status) ”方法来定义需要事务管理的操作代码；
TransactionCallbackWithoutResult：继承TransactionCallback接口，提供“void doInTransactionWithoutResult(TransactionStatus status)”便利接口用于方便那些不需要返回值的事务操作代码。

```java
@Test
public void testTransactionTemplate(){
	jdbcTemplate = new JdbcTemplate(dataSource);
    int i = jdbcTemplate.queryForInt(COUNT_SQL);  
    System.out.println("表中记录总数："+i);
	//构造函数初始化TransactionTemplate
	TransactionTemplate template = new TransactionTemplate(txManager);
	template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);  
	//重写execute方法实现事务管理
	template.execute(new TransactionCallbackWithoutResult() {
		@Override
		protected void doInTransactionWithoutResult(TransactionStatus status) {
			jdbcTemplate.update(INSERT_SQL, "饿死");   //字段sd为int型，所以插入肯定失败报异常，自动回滚，代表TransactionTemplate自动管理事务
		}}
	);
	i = jdbcTemplate.queryForInt(COUNT_SQL);  
    System.out.println("表中记录总数："+i);
}
```

## 3.声明式事务：可知编程式事务每次实现都要单独实现，但业务量大功能复杂时，使用编程式事务无疑是痛苦的，而声明式事务不同，声明式事务属于无侵入式，不会影响业务逻辑的实现。

声明式事务实现方式主要有2种，一种为通过使用Spring的<tx:advice>定义事务通知与AOP相关配置实现，另为一种通过@Transactional实现事务管理实现，下面详细说明2种方法如何配置，已经相关注意点

### 1)方式一，配置文件如下

```java
<!-- 
<tx:advice>定义事务通知，用于指定事务属性，其中“transaction-manager”属性指定事务管理器，并通过<tx:attributes>指定具体需要拦截的方法
	<tx:method>拦截方法，其中参数有：
	name:方法名称，将匹配的方法注入事务管理，可用通配符
	propagation：事务传播行为，
	isolation：事务隔离级别定义；默认为“DEFAULT”
	timeout：事务超时时间设置，单位为秒，默认-1，表示事务超时将依赖于底层事务系统；
	read-only：事务只读设置，默认为false，表示不是只读；
    rollback-for：需要触发回滚的异常定义，可定义多个，以“，”分割，默认任何RuntimeException都将导致事务回滚，而任何Checked Exception将不导致事务回滚；
    no-rollback-for：不被触发进行回滚的 Exception(s)；可定义多个，以“，”分割；
 -->
<tx:advice id="advice" transaction-manager="transactionManager">
	<tx:attributes>
	    <!-- 拦截save开头的方法，事务传播行为为：REQUIRED：必须要有事务, 如果没有就在上下文创建一个 -->
		<tx:method name="save*" propagation="REQUIRED" isolation="READ_COMMITTED" timeout="" read-only="false" no-rollback-for="" rollback-for=""/>
		<!-- 支持,如果有就有,没有就没有 -->
		<tx:method name="*" propagation="SUPPORTS"/>
	</tx:attributes>
</tx:advice>
<!-- 定义切入点，expression为切人点表达式，如下是指定impl包下的所有方法，具体以自身实际要求自定义  -->
<aop:config>
    <aop:pointcut expression="execution(* com.kaizhi.*.service.impl.*.*(..))" id="pointcut"/>
    <!--<aop:advisor>定义切入点，与通知，把tx与aop的配置关联,才是完整的声明事务配置 -->
    <aop:advisor advice-ref="advice" pointcut-ref="pointcut"/>
</aop:config>
```

注意点：
事务回滚异常只能为RuntimeException异常，而Checked Exception异常不回滚，捕获异常不抛出也不会回滚，但可以强制事务回滚：TransactionAspectSupport.currentTransactionStatus().isRollbackOnly();
解决“自我调用”而导致的不能设置正确的事务属性问题，可参考http://www.iteye.com/topic/1122740

### 2）方式二通过@Transactional实现事务管理

```java
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">   
     <property name="dataSource" ref="dataSource"/>
</bean>    
<tx:annotation-driven transaction-manager="txManager"/> //开启事务注解
```

@Transactional(propagation=Propagation.REQUIRED,isolation=Isolation.READ_COMMITTED)，具体参数跟上面<tx:method>中一样
Spring提供的@Transaction注解事务管理，内部同样是利用环绕通知TransactionInterceptor实现事务的开启及关闭。
使用@Transactional注意点：

1、如果在接口、实现类或方法上都指定了@Transactional 注解，则优先级顺序为方法>实现类>接口；
2、建议只在实现类或实现类的方法上使用@Transactional，而不要在接口上使用，这是因为如果使用JDK代理机制（基于接口的代理）是没问题；而使用使用CGLIB代理（继承）机制时就会遇到问题，因为其使用基于类的代理而不是接口，这是因为接口上的@Transactional注解是“不能继承的”；

