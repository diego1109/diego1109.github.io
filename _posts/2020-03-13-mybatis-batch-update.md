---

layout: post
title: mybatis批量更新
date: 2020-03-13
categories: 学习笔记
tags: Mybatis
comments: true 

---

工作中遇到个场景需要批量更新表中的记录。当时很明确了，在业务代码里使用循环更新这种方式肯定不可取的，频繁连接数据库是个糟糕的设计。所以就想到批量更新，只连接一次数据库，逐条更新的操作放到数据库里面去执行。即便已经意识到了多次连接会造成效率下降，但还需要对场景再考虑下，选择适合的逻辑实现方式。

# 使用场景

在需要批量更新时，就查询条件相对于更新的对象而言，会有**一对多**和**多个一对一**两种的场景。
场景1：老师要给全班所有学生的数学成绩加10分，用老师Id作为查询条件就可以筛选出全班所有的同学，“每个同学的数学成绩加10分”表示所有查询出来的记录要更新的值是一样的。
场景2：某次考试结束了，老师要更新每个同学的数学成绩，这种场景下老师Id就不可能在作为查询条件了，送入的参数应该是个Student列表，每个Student对象中包含学生Id和他的数学成绩，SQL通过for循环的方式更新每个同学的成绩。

在动手之前想清楚当下问题复合哪种场景，那基本的架子就算是有。接下来就是细节上的事情了，该送入什么参数，具体怎么更新....等等。等这些都确认好，实现已经不难了。

# 一对多

这种批量更新跟简单，而且特征也很明显：（1）确定的查询条件（可以是and组合的）能够筛选出所有要更新的记录；（2）这些记录要更新的值或者关键值是一样的。就如同场景1，代码应该如下：

```xml
<update id="updateMath">
  update student
  set math_score = math_score + #{addMathScore}
  where teacher_id = #{teacherId}
</update>
```

`addMathScore` 和 `teacherId` 由 mapper.xml 中的接口送进来就可以了。更新数学成绩的场景很简单，但我觉得还是需要再推敲下的。采用“多个一对一”的方式给每个学生的数学成绩加10分也是可行的，而且脑子里首先想到的应该也是这种方式。但要真这么干反而麻烦了（先不说两种执行方式效率的快慢），首先得构建个类(student.class)，再生成多个 student object ，这些object 中的 `addMathScore` 是一样的，`studentId` 各不相同，这就有种很“浪费”的感觉。也许会想到：可以用单独的变量保存 `addMathScore` ，再定义一个 `List<String>` 变量存储所有的 `studentId` ，这样就简单好多了。但等真的动手的时候又会发现，所有的学生Id又得先从数据库中搂出来。其实在这种场景下，从 class 到 list object生成远没有两个变量来的简单。

在工作中会遇到批量修改名字或者修改某些字段的需求，开发的之前应该主动想一下当前问题否适合这一场景，因为它简洁明了却“藏”在后面。



# 多个一对一

与上面的对应，这种场景的特征是：（1）唯一确定的where条件只能选出一条记录；（2）而且这些记录更新的值每个都是不一样的。这样只能将 **筛选条件** 和 **要更新的值**封装在对象中，以list object的形式送到MySQL中，最后通过循环的形式更新每条记录。

## 方式一

```xml
<update id="updateBatch"  parameterType="java.util.List">  
    <foreach collection="list" item="item" index="index" open="" close="" separator=";">
        update tableName
        <set>
            name=#{item.name},
            name2=#{item.name2}
        </set>
        where id = #{item.id}
    </foreach>      
</update>
```

先贴上代码，这种更新简洁明了，使用for循环一条记录update一次，但效率不是很高。注意的是：MySQL 默认不支持多条 SQL 语句执行，所以需要在 MySQL 的 URL 后面添加 `&allowMultiQueries=true` 才能保证上述方式运行成功。还有一点要注意的是，H2 不支持这样的操作，公司项目里本地测境用的是 H2，线上用的是 MySQL，着实被这块小坑了一把。



## 方式二

```xml
<update id="list"  parameterType="java.util.List">
  update tableName
  <trim prefix="set" suffixOverrides=",">
    <trim prefix="owner_id =case" suffix="end,">
      <foreach collection="list" item="item" index="index">
        when full_id=#{item.originalFullId} then #{item.newOwnerId}
      </foreach>
    </trim>
    <trim prefix="full_id =case" suffix="end,">
      <foreach collection="list" item="item" index="index">
        when full_id=#{item.originalFullId} then #{item.newFullId}
      </foreach>
    </trim>
  </trim>
  where full_id in
  <foreach collection="list" index="index" item="item" separator="," open="(" close=")">
    #{item.originalFullId,jdbcType=VARCHAR}
  </foreach>
</update>
```

这种方式使用的SQL的 `case-when-then` 语法实现了批量更新，H2和MySQL都可以执行。

SQL语法原型：

```mysql
 UPDATE course
    SET name = CASE id 
        WHEN 1 THEN 'name1'
        WHEN 2 THEN 'name2'
        WHEN 3 THEN 'name3'
    END, 
    title = CASE id 
        WHEN 1 THEN 'New Title 1'
        WHEN 2 THEN 'New Title 2'
        WHEN 3 THEN 'New Title 3'
    END
WHERE id IN (1,2,3)
```

这条sql的意思是，如果id为1，则name的值为name1，title的值为New Title1；依此类推，将所有的cast都给摆出来，对号入座。

mybatis拼出来的结果：

```mysql
update mydata_table 
	set status = 
case
	when id = #{item.id} then #{item.status} end,
	...
	when id = id = #{item.id} then #{item.status} end
	where id in (...);
```

trim标签：

```xml
<trim prefix="" suffix="" suffixOverrides="" prefixOverrides=""></trim>
```

 `prefix:` 如果 trim 中有内容，则在 SQL 语句中加上 prefix 指定的字符串前缀。
`prefixOverrides:` 如果 trim 中有内容，去除 prefixOverrides 指定的多余的前缀内容。
`suffix:` 如果 trim 中有内容，则在 SQL 语句中加上 suffix 指定的字符串后缀。
`suffixOverrides`: 如果 trim 中有内容，去除 suffixOverrides 指定的多余的后缀内容。



**这里有个要注意的地方：**

方式二更新了两个字段: owner_id 和 full_id ，当数据库是 MySQL 时，这两个字段中谁是主键，则这个字段要放在最后更新，不然会更新失败。但如果数据库是 H2 ，则没有这个顺序要求。有点疑惑~~~



## 方式三

上述两种方法都是在拼SQL，[他们被一些开发者吐槽是奇技淫巧](https://blog.csdn.net/w605283073/article/details/83064000)。至于到底什么事奇技淫巧好像没有定义啊，我觉得还是得从场景、效率、可读性来评价当前实现方式的优劣。

mybatis对批量更新提供了正确打开方式：[ExecutorType.BATCH](https://github.com/mybatis/mybatis-3/blob/master/src/test/java/org/apache/ibatis/submitted/batch_keys/BatchKeysTest.java)。

这种方式不适合XML格式的mybatis操作。

# 总结

“磨刀不误砍柴工”，敲代码已经是最后一道工序了，但在动手敲之前需要先想清楚实现功能的代码架子是什么样子，将有疑惑的细节确认清楚，这个很重要。这些都想的差不多了，敲代码就会有底气，效率也会高起来。