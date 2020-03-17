---

layout: post
title: mybatis批量更新
date: 2020-03-13
categories: 学习笔记
tags: Mybatis
comments: true 

---

工作中遇到个场景需要批量更新表中的记录。当时很明确了，在业务代码里使用循环更新这种方式肯定不可取的，频繁连接数据库是个糟糕的设计。所以就想到批量更新，只连接一次数据库，逐条更新的操作放到数据库里面去执行。中途遇到了些小问题，不过都解决了。

# 方式一

```xml
<update id="updateBatch"  parameterType="java.util.List">  
    <foreach collection="list" item="item" index="index" open="" close="" separator=";">
        update tableName
        <set>
            name=${item.name},
            name2=${item.name2}
        </set>
        where id = ${item.id}
    </foreach>      
</update>
```

先贴上代码，这种更新简洁明了，不在多做解释。不过要注意的是：MySQL 默认不支持多条 SQL 语句执行，所以需要在 MySQL 的 URL 后面添加 `&allowMultiQueries=true` 才能保证上述方式运行成功。还有一点要注意的是，H2 不支持这样的操作，公司项目里本地测境用的是 H2，线上用的是 MySQL，着实被这块小坑了一把。

# 方式二

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

这种方式的批量更新就不需要修改 URL 了。

```xml
<trim prefix="" suffix="" suffixOverrides="" prefixOverrides=""></trim>
```

 `prefix:` 如果 trim 中有内容，则在 SQL 语句中加上 prefix 指定的字符串前缀。

`prefixOverrides:` 如果 trim 中有内容，去除 prefixOverrides 指定的多余的前缀内容。

 `suffix:` 如果 trim 中有内容，则在 SQL 语句中加上 suffix 指定的字符串后缀。

`suffixOverrides`: 如果 trim 中有内容，去除 suffixOverrides 指定的多余的后缀内容。



**这里有个要注意的地方：**

方式二更新了两个字段: owner_id 和 full_id ，当数据库是 MySQL 时，这两个字段中谁是主键，则这个字段要放在最后更新，不然会更新失败。但如果数据库是 H2 ，则没有这个顺序要求。有点疑惑~~~



# 方式三

上述两种方法都是在拼SQL，[他们被一些开发者吐槽是奇技淫巧](https://blog.csdn.net/w605283073/article/details/83064000)。

mybatis对批量更新提供了正确打开方式：[ExecutorType.BATCH](https://github.com/mybatis/mybatis-3/blob/master/src/test/java/org/apache/ibatis/submitted/batch_keys/BatchKeysTest.java)。