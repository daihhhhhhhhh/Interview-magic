# mongoDB的索引详解

## 1 mongoDB索引的管理

　　本节介绍mongoDB中的索引，熟悉mysql/sqlserver等关系型数据库的小伙伴应该都知道索引对优化数据查询的重要性。我们先简单了解一下索引：索引的本质就是一个排序的列表，在这个列表中存储着索引的值和包含这个值的数据(数据row或者document)的物理地址，索引可以大大加快查询的速度，这是因为使用索引后可以不再扫描全表来定位某行的数据，而是先通过索引表找到该行数据对应的物理地址(多数为B-tree查找)，然后通过地址来访问相应的数据。

　　索引可以加快数据检索、排序、分组的速度，减少磁盘I/O，但是索引也不是越多越好，因为索引本身也是数据表，需要占用存储空间，同时索引需要数据库进行维护，当我们对索引列的值进行增改删操作时，数据库需要更新索引表，这会增加数据库的压力。

我们要根据实际情况来判断哪些列适合添加索引，哪些列不适合添加索引，一般遵循的规律如下：

　　主/外键列，主键用于强制该列的唯一性和组织表中数据的排列结构；外键可以加快连接的速度；

　　经常用于比较的类(大于小于等于等)，因为索引已经排序，值就是大于/小于的分界点；

　　经常进行范围搜索，因为索引已经排序，其指定的范围是连续的；

　　经常进行排序的列，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间；

　　经常进行分组的列，因为索引已经排序，同一个值的所有数据地址会聚集在一块，很方便分组。

我们看一下mongoDB的索引使用，首先准备数据：



```
db.userinfos.insertMany([
   {_id:1, name: "张三", age: 23,level:10, ename: { firstname: "san", lastname: "zhang"}, roles: ["vip","gen" ]},
   {_id:2, name: "李四", age: 24,level:20, ename: { firstname: "si", lastname: "li"}, roles:[ "vip" ]},
   {_id:3, name: "王五", age: 25,level:30, ename: { firstname: "wu", lastname: "wang"}, roles: ["gen","vip" ]},
   {_id:4, name: "赵六", age: 26,level:40, ename: { firstname: "liu", lastname: "zhao"}, roles: ["gen"] },
   {_id:5, name: "田七", age: 27, ename: { firstname: "qi", lastname: "tian"}, address:'北京' },
   {_id:6, name: "周八", age: 28,roles:["gen"], address:'上海' }
]);  
```



　　索引的增删改查还是十分简单的，我们看一下索引管理的几个方法：



```
//创建索引,值1表示正序排序，-1表示倒序排序
　　db.userinfos.createIndex({age:-1})

//查看userinfos中的所有索引
　　db.userinfos.getIndexes()

//删除特定一个索引
　　db.userinfos.dropIndex({name:1,age:-1})
//删除所有的索引(主键索引_id不会被删除)
　　db.userinfos.dropIndexes()

//如果我们要修改一个索引的话，可以先删除索引然后在重新添加。 
```





## 2 mongoDB中常用的索引类型



### 1 单键索引

　　单键索引(Single Field Indexes)顾名思义就是单个字段作为索引列，mongoDB的所有collection默认都有一个单键索引_id，我们也可以对一些经常作为过滤条件的字段设置索引，如给age字段添加一个索引，语法十分简单：

```
//给age字段添加升序索引
　　db.userinfos.createIndex({age:1})
```

　　其中{age:1}中的1表示升序，如果想设置倒序索引的话使用 db.userinfos.createIndex({age:-1}) 即可。我们通过explain()方法查看查询计划，如下图，看到查询age=23的document时使用了索引，如果没有使用索引的话stage=COLLSCAN。

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190616162439895-28686184.png)

　　因为document的存储是bson格式的，我们也可以给内置对象的字段添加索引，或者将整个内置对象作为一个索引，语法如下：



```
//1.内嵌对象的某一字段作为索引
//在ename.firstname字段上添加索引
　　db.userinfos.createIndex({"ename.firstname":1})
//使用ename.firstname字段的索引查询
　　db.userinfos.find({"ename.firstname":"san"})

//2.整个内嵌对象作为索引
//给整个ename字段添加索引
　　db.userinfos.dropIndexes()
//使用ename字段的索引查询
　　db.userinfos.createIndex({"ename":1})
```





### 2 复合索引

　　复合索引(Compound Indexes)指一个索引包含多个字段，用法和单键索引基本一致。使用复合索引时要注意字段的顺序，如下添加一个name和age的复合索引，name正序，age倒序，document首先按照name正序排序，然后name相同的document按age进行倒序排序。mongoDB中一个复合索引最多可以包含32个字段。



```
//添加复合索引，name正序，age倒序
  　　db.userinfos.createIndex({"name":1,"age":-1}) 
//过滤条件为name，或包含name的查询会使用索引(索引的第一个字段)
  　　db.userinfos.find({name:'张三'}).explain()
　　  db.userinfos.find({name:"张三",level:10}).explain()
　　  db.userinfos.find({name:"张三",age:23}).explain()

//查询条件为age时，不会使用上边创建的索引,而是使用的全表扫描
db.userinfos.find({age:23}).explain()
```



　　执行查询时查询计划如下：

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190616165236071-2077681014.png)



### 3 多键索引

　　多键索引(mutiKey Indexes)是建在数组上的索引，在mongoDB的document中，有些字段的值为数组，多键索引就是为了提高查询这些数组的效率。看一个栗子：准备测试数据，classes集合中添加两个班级，每个班级都有一个students数组，如下：



```
  db.classes.insertMany([
     {
         "classname":"class1",
         "students":[{name:'jack',age:20},
                    {name:'tom',age:22},
                    {name:'lilei',age:25}]
      },
      {
         "classname":"class2",
         "students":[{name:'lucy',age:20},
                    {name:'jim',age:23},
                    {name:'jarry',age:26}]
      }]
  )
```



　　为了提高查询students的效率，我们使用 db.classes.createIndex({'students.age':1}) 给students的age字段添加索引，然后使用索引，如下图：

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190616181241314-1941755772.png)



###  4 哈希索引

　　哈希索引(hashed Indexes)就是将field的值进行hash计算后作为索引，其强大之处在于实现O(1)查找，当然用哈希索引最主要的功能也就是实现定值查找，对于经常需要排序或查询范围查询的集合不要使用哈希索引。

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190616184350043-240369758.png)



## 3 mongoDB中常用的索引属性



### 1 唯一索引

　　唯一索引(unique indexes)用于为collection添加唯一约束，即强制要求collection中的索引字段没有重复值。添加唯一索引的语法：

```
//在userinfos的name字段添加唯一索引
db.userinfos.createIndex({name:1},{unique:true})
```

　　看一个使用唯一索引的栗子：

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190617224005485-827258022.png)



### 2 局部索引

　　局部索引(Partial Indexes)顾名思义，只对collection的一部分添加索引。创建索引的时候，根据过滤条件判断是否对document添加索引，对于没有添加索引的文档查找时采用的全表扫描，对添加了索引的文档查找时使用索引。使用方法也比较简单：



```
//userinfos集合中age>25的部分添加age字段索引
    db.userinfos.createIndex(
        {age:1},
        { partialFilterExpression: {age:{$gt: 25 }}}
    )
//查询age<25的document时，因为age<25的部分没有索引，会全表扫描查找(stage:COLLSCAN)
    db.userinfos.find({age:23})
//查询age>25的document时，因为age>25的部分创建了索引，会使用索引进行查找(stage:IXSCAN)
    db.userinfos.find({age:26})
```



　　当查询age=23的记录时，stage=COLLSCAN，当查询age=26的记录时，使用了索引，如下：

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190616191305734-1488348448.png)



### 2 稀疏索引

　　稀疏索引(sparse indexes)在有索引字段的document上添加索引，如在address字段上添加稀疏索引时，只有document有address字段时才会添加索引。而普通索引则是为所有的document添加索引，使用普通索引时如果document没有索引字段的话，设置索引字段的值为null。

　　稀疏索引的创建方式如下，当document包含address字段时才会创建索引：

```
//创建在address上创建稀疏索引
　　db.userinfos.createIndex({address:1},{sparse:true})
```

　　看一个使用稀疏索引的栗子：

![img](https://img2018.cnblogs.com/blog/1007918/201906/1007918-20190617204220128-694064785.png)



### 4 TTL索引

　　TTL索引(TTL indexes)是一种特殊的单键索引，用于设置document的过期时间，mongoDB会在document过期后将其删除，TTL非常容易实现类似缓存过期策略的功能。我们看一个使用TTL索引的栗子：



```
 //添加测试数据
db.logs.insertMany([
       {_id:1,createtime:new Date(),msg:"log1"},
       {_id:2,createtime:new Date(),msg:"log2"},
       {_id:3,createtime:new Date(),msg:"log3"},
       {_id:4,createtime:new Date(),msg:"log4"}
       ])
       //在createtime字段添加TTL索引，过期时间是120s
       db.logs.createIndex({createtime:1}, { expireAfterSeconds: 120 })


//logs中的document在创建后的120s后过期，会被mongoDB自动删除
```



　　注意：TTL索引只能设置在date类型字段(或者包含date类型的数组)上，过期时间为字段值+exprireAfterSeconds；document过期时不一定就会被立即删除，因为mongoDB执行删除任务的时间间隔是60s；capped Collection不能设置TTL索引，因为mongoDB不能主动删除capped Collection中的document。

**小结**

　　本节介绍了mongoDB中常用的索引和索引属性，索引对提升数据检索的速度十分重要，在数据量比较大的时候一般都要在collection上建立索引。mongoDB提供的索引种类很丰富，总会有几种适用于我们的业务，除了上边介绍的索引外，mongoDB还支持text index和一些地理位置相关的索引，这里不再介绍，有兴趣的小伙伴可以到[官网](https://docs.mongodb.com/manual/indexes/) 研究下。如果文中有错误的话，希望大家可以指出，我会及时修改，谢谢。