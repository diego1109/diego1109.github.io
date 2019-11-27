---
layout: post
title:  "Mybatis select 查询返回值为List集合+源码"
date:   2019-11-27 20:04:09 +0800
categories: Mybatis
---

对mysql做查询的时候返回值经常是个`List<object>`，在mapper.xml对应的statement中有时候是`resultType`有时候却是`resultMap`。傻傻分不清楚所以总是去找以前的代码看看是怎么写的，等花时间整理的这块的时候才发现用哪个其实很简单，几分钟就能理清楚。[博客中提到的查询场景在源码的测试中都有对应](https://pan.baidu.com/s/1UG9YPDohEFBPtP-3fnOpOQ)。

假设我现在要查询文章，`Article`类有三个属性：文章的id，文章所属的用户id，文章内容。

```java
public class Article {
  private String id;
  private String userId;
  private String body;

  public Article(String userId, String body) {
    this.id = UUID.randomUUID().toString();
    this.userId = userId;
    this.body = body;
  }
}
```

场景一：我要查询一个用户所有文章的Id，很明显查询的结果是个`List<String>`,别忘了string也是个`Object`。此时用`resultType`，先这么记着下面再做解释，对应的statement如下所示：

```xml
<select id="findArticles" resultType="java.lang.String">
  select id
  	from articles A
  where A.user_id = #{userId}
</select>
```

场景二：我要查询一个用户的所有文章，不着急根据读写分离的原则我们先定义一个专门用来接收读取的文章的类`ArticleData`，它的属性跟`Article`一模一样。

```java
public class ArticleData {
  private String id;
  private String userId;
  private String body;
}
```

这个查询结果也很明显肯定是个`List<ArticleData>`，但好多人困惑的是“老虎”、“老鼠”到底用哪个。先用`resultMap`试试，对应的statement如下所示，给要查的字段取个别名如`A.id`的别名是`articleId`，最后在resultMap中将字段名和ArticleData的属性一一映射起来，这样查询就完成了。具体的Mybatis查询语法这里就不解释了。

```xml
<sql id="articleData">
  select
    A.id articleId,
    A.user_id articleUserId,
    A.body articleBody
</sql>

<select id="findByUserId" resultMap="articleData">
  <include refid="articleData"/>
  from articles A
  where A.user_id = #{userId}
</select>

<resultMap id="articleData" type="com.yang.mybatis.application.data.ArticleData">
  <id property="id" column="articleId"/>
  <result property="userId" column="articleUserId"/>
  <result property="body" column="articleBody"/>
</resultMap>
```

那能否用`resultType`呢？答案是可以的。只是有一个要求：被查询的字段的名字必须和接收对象的属性名一样。看个呆萌就就明白什么意思了。statuement得这么写：

```xml
<select id="findByUserId" resultType="com.yang.mybatis.application.data.ArticleData">
  select
    A.id id,
    A.user_id userId,
    A.body body
  from articles A
  where A.user_id = #{userId}
</select>
```

三个待查字段取的别名跟`ArticleData`中属性的名字一样，否则这种方式就会报错。再回头看下场景一的查询，是因为`String`能匹配任意字符串。