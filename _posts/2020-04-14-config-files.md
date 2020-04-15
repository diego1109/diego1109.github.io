---

layout: post
title: spring boot 属性注入
date: 2020-04-15
categories: 学习笔记
tags: SpringBoot
comments: true 

---



## 不需要配置文件场景下的属性注入

 @Value注解可以在不需要配置文件的情况下注入属性，经常会看到两种写法 `${}`和 `#{}` ：

`${properties名称}`的表示获取properties的值，注入配置文件中的属性的值。

 `#{表达式}`  括号里面的表达式必须是[SpEL表达式的格式](https://docs.spring.io/spring/docs/5.2.5.RELEASE/spring-framework-reference/core.html#expressions-beandef-annotation-based)。通过 `#{表达式}` 方式可以在不需要使用配置文件的情况下，动态的将外部值注入到bean中。 

````java
@Component
@ToString
public class InjectWithoutConfigurationFile {

  // 注入普通字符串
  @Value("diego")
  private String name;

  // 注入表达式
  @Value("#{T(java.lang.Math).random()}")
  private double random;

  // 注入系统属性
  @Value("#{systemProperties['os.name']}")
  private String defaultLocale;

  // 注入其他bean的属性值
  @Value("#{rectangle.length}")
  private int rectangleLength;

  // 注入url
  @Value("http://www.baidu.com")
  private Resource baiduUrls;

}
````

测试结果：

````java
InjectWithoutConfigurationFile(name=diego, random=0.20947710881441184, defaultLocale=Mac OS X, rectangleLength=20, baiduUrls=URL [http://www.baidu.com])
````

## 配置文件中的属性注入

在Spring Boot项目中，先在全局配置文件中配置好属性值，在项目启动时这些属性自动注入到对应bean对应属性中。

全局配置文件：

- application.properties
- application.yaml

这两个都是全局配置文件，但properties的优先级高于yaml，当两个文件中对统一属性配置不同的值时，以properties文件中的为准，两个文件中的不同配置互补。

实现配置文件中的属性自动注入有一下几种方式：

### @Value的${}方式

````properties
value.from.file=Value got from the file
priority=Properties file
listOfValues=A,B,C
````

````java
@ToString
@Component
public class Properties {

  @Value("${value.from.file}")
  private String valueFromFile;
  
  @Value("${priority}")
  private String prioritySystemProperty;

  @Value("${listOfValues}")  
  private String[] valuesArray; // 注入数组
}
````

在测试中如果将listOfValues卸载yaml配置文件中，形如下面的样子，再用@Value("${listOfValues}") 注入的时候会报错。

````yml
listOfValues：
	- A
	- B
	- C
````

**在开发过程中，如果需要将配置文件中的单个属性注入bean中，采用@Value是最简单的方式。当然@Value注解也可以将object注入，但总觉得操作有限。**

### @ConfigurationProperties

当配置文件中的object很复杂时，选用@ConfigurationProperties将object注入会很方便。@ConfigurationProperties注解的使用也有好几种方式，这里主要介绍两种：

1. Spring Boot 2中，可以使用@ConfigurationProperties和@ConfigurationPropertiesScan两个注解实现配置注入。

    ````yaml
    mail:
      hostname: host@mail.com
      port: 9000
      from: mailer@mail.com
      defaultRecipients:
        - admin@mail.com
        - owner@mail.com
      additionalHeaders:
        redelivery: true
        secure: true
        p3: value
    ````

    ````java
    @SpringBootApplication
    @ConfigurationPropertiesScan("com.yang.config")
    public class ConfigApplication {
    
      public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
      }
    }
    ````

    ```java
    @ConfigurationProperties(prefix = "mail")
    @ToString
    @Setter
    public class ConfigProperties {
    
      private String hostName;
      private int port;
      private String from;
    
      private List<String> defaultRecipients;
      private Map<String, String> additionalHeaders;
    
    }
    ```

    @ConfigurationPropertiesScan("com.yang.config")表示扫描com.yang.config包下的@ConfigurationProperties注解，@ConfigurationProperties(prefix = "mail")表示将配置文件中前缀为mail的属性与ConfigProperties中的对应的成员变量映射起来。必须为要映射的成员变量创建setter方法，才能将配置文件中的属性值映射成功。

    测试结果：

    ````java
    ConfigProperties(hostName=host@mail.com, port=9000, from=mailer@mail.com, defaultRecipients=[admin@mail.com, owner@mail.com], additionalHeaders={redelivery=true, secure=true, p3=value})
    ````

2. 使用@ConfigurationProperties和@Component注解

    ````java
    @Component
    @ConfigurationProperties(prefix = "mail")
    @ToString
    @Setter
    public class ConfigProperties {
    
      private String hostName;
      private int port;
      private String from;
    
      private List<String> defaultRecipients;
      private Map<String, String> additionalHeaders;
    
    }
    ````

    当然还有其他的的注解也能实现复杂的object的注入，我觉得上面这两种使用简单，理解也很容易。

### @PropertySource

有时候我们可能会自定义配置文件，专门用来保存某个类的配置，这时需要使用@PropertySource注解先将自定义配置文件加载进来。

````properties
# customConfigProperties.properties配置文件

#Simple properties
custom.hostname=host@mail.com
custom.port=9000
custom.from=mailer@mail.com

#List properties
custom.defaultRecipients[0]=admin@mail.com
custom.defaultRecipients[1]=owner@mail.com

#Map Properties
custom.additionalHeaders.redelivery=true
custom.additionalHeaders.secure=true
custom.additionalHeaders.p3=value
````

```java
@PropertySource("classpath:/customConfigProperties.properties")
@ConfigurationProperties(prefix = "custom")
@Component
@ToString
@Setter
public class CustomConfigProperties {

  private String hostName;
  private int port;
  private String from;

  private List<String> defaultRecipients;
  private Map<String, String> additionalHeaders;
}
```

可以看出@PropertySource注解将自定义的customConfigProperties.properties配置文件加载进来。之后用@ConfigurationProperties和@Component组合的方式将配置文件的中的object注入。

**注：@PropertySource注解不支持加载自定义yaml格式的配置文件。**

测试结果：

````java
CustomConfigProperties(hostName=host@mail.com, port=9000, from=mailer@mail.com, defaultRecipients=[admin@mail.com, owner@mail.com], additionalHeaders={redelivery=true, secure=true, p3=value})
````



### 实现EnvironmentPostProcessor

