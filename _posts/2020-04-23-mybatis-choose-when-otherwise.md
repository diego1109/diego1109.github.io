---

layout: post
title: mybatis 使用 choose when otherwise 规避异常  
date: 2020-04-23
categories: 学习笔记
tags: Mybatis
comments: true 

---

也是在工作中遇到小问题，查到原因后依旧是没有规避到空值。

### 简化的场景

假设我们有一张学生表：

````mysql
create table students(
	name varchar(255) not null,
    sex varchar(255) not noll,
    score int(11);
)
````

这张表也很简单，就姓名、性别、成绩三个字段。

也是很简单的一个场景，但能说明问题。

需求：将一些学生的成绩减去10分。

````xml
<update id="updateStudentScore">
    update student
    set core = score - 10
    where name in 
    <foreach collection="names="item" separator="," open="(" close=")">
      #{item}
    </foreach>
</update>
````

这种写法正常情况时完全ok的，但是当names是个空列表时，就会报异常。

### 使用 choose when otherwise

````xml
<update id="updateStudentScore">
    update student
    set core = score - 10
    where name in 
    <choose>
      <when test="names == null || names.isEmpty()">
        where 1=0
      </when>
      <otherwise>
        where name in
        <foreach collection="names" index="index" item="item" separator="," open="(" close=")">
          #{item}
        </foreach>
      </otherwise>
    </choose>
</update>
````

这种动态SQL就可以规避掉空列表异常的现象，类似于java的 `if..else if..else`。当然也可以在外围判断类表是否为空，如果是，则不执行update score操作，这也是可以的，当然，最终是要根据尝尽确定解决办法。

