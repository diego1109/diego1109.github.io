---
layout: post
title: 基于flyway的spring boot migration
date: 2020-02-23
categories: 学习笔记
tags: SpringBoot
comments: true 
---

**Flyway**是个开源的数据库迁移工具（把migration翻译成迁移感觉听不顺嘴的，暂且这么叫吧）。它提供了 `SQL`和 `JAVA`两种方式做migration，并且使用也很简单。基于Spring Boot的应用开发场景，本文内容:

### 一. 添加依赖和配置

build.gradle：

```json
dependencies{
	implementation('org.flywaydb:flyway-core')
}
```

application.properties:

```properties
spring.datasource.url=jdbc:mysql://localhost/demo_database_1
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
debug=true

flyway.baseline-on-migrate=true #used if database has some already table
flyway.enabled=true 
flyway.url=jdbc:mysql://localhost/demo_database_1
flyway.user=root
flyway.password=
```

### 二. 使用方法

sql方式的migration需要创建一个 `.sql` 文件，sql方式一般用来创建表、删除表，字段的添加、删除或者修改字段的名字、类型和初值等。如要想要对标中已有的数据执行逻辑操作，那就需要采用java的方式做migration，同样也需要创建 `.java` 文件。在spring boot程序启动的时候，flyway会按照顺序执行migration文件。

#### 2.1migration文件的命名和放置的地方

```java
V1__Add_new_table
```

其中：

- **V** 是migration版本的前缀;
- **1** 是版本号，从1往上增加，不能重复;
- **Add_new_table** 是文件名;

.sql文件放在 `src/main/resources/db/migration` 路径下；

.java文件放在 `src/main/java/db/migration`路径下；

#### 2.2 SQL方式的migration

直接写SQL语句就可以了，很简单的。场景：创建一个employee表。

创建文件：resources/db/migration/V1__Create_Employee_Table.sql

SQL语句：

```sql
CREATE TABLE `employee` (
  `employeeId` int(11) NOT NULL AUTO_INCREMENT,
  `employeeName` varchar(255) DEFAULT NULL,
  `employeeRole` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`employeeId`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT;
```

#### 2.3 JAVA方式的migration

创建的java文件中必须实现 `SpringJdbcMigration` 接口，在重写的 `migration`方法中编写业务逻辑。

场景1：查询employee中的员工数目。

```java
package db.migration;

import org.flywaydb.core.api.migration.spring.SpringJdbcMigration;
import org.springframework.jdbc.core.JdbcTemplate;

public class V4__Another_user implements SpringJdbcMigration{
    @Override
    public void migrate(JdbcTemplate jdbcTemplate) throws Exception {
        int result = jdbcTemplate.queryForObject(
    "SELECT COUNT(*) FROM EMPLOYEE", Integer.class);
    }
}
```

场景2：查询所有员工的名字。（查询单个字段）

```java
List<String> name = jdbcTemplate.queryForList(
        "select distinct(employeeId) from employee", String.class);
```

场景3：查询所有员工的ID、name和role。（查询多个字段）

用 `RowMapper`接口将查询结果映射到java对象。

```java
public class EmployeeRowMapper implements RowMapper<Employee> {
    @Override
    public Employee mapRow(ResultSet rs, int rowNum) throws SQLException {
        Employee employee = new Employee();
 
        employee.setId(rs.getInt("id"));
        employee.setName(rs.getString("name"));
        employee.setAddress(rs.getString("role"));
 
        return employee;
    }
}

@Data
class Employee{
  int id;
  String name;
  String role;
}
```

对应的migration方法中的查询语句如下：

```java
String query = "SELECT employeeId as id, employeeName as name, employeeRole as role FROM EMPLOYEE";
List<Employee> employees = jdbcTemplate.queryForObject(
  query, new EmployeeRowMapper());
```

场景4：查询姓名是“diego”的员工的信息。（带参查询）

```java
String query = "SELECT * FROM EMPLOYEE WHERE employeeName = ?";
Employee employee = jdbcTemplate.queryForObject(
  query, new Object[] { “diego” }, new EmployeeRowMapper());
```

场景5：将所有员工的ID在原有基础上加1，这个逻辑操作很简单，但在实际应用中会出现很复杂的逻辑操作。（批量更新）

```java
//获取所有员工信息。
String query = "SELECT employeeId as id, employeeName as name, employeeRole as role FROM EMPLOYEE";
List<Employee> employees = jdbcTemplate.queryForObject(
  query, new EmployeeRowMapper());

//更新
String updateUserInfoSQL = "update EMPLOYEE set employeeId = ? where employeeName = ?";
    jdbcTemplate.batchUpdate(updateUserInfoSQL, new BatchPreparedStatementSetter() {
      @SneakyThrows
      @Override
      public void setValues(PreparedStatement ps, int i) throws SQLException {
        Employee employee = employees.get(i);
        ps.setString(1, employee.getId()+1);
        ps.setString(2, employee.getName());
      }

      @Override
      public int getBatchSize() {
        return employees.size();
      }
    });
```

