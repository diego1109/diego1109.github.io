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

有时候我们可能会**自定义配置文件**，专门用来保存某个类的配置，这时需要使用@PropertySource注解先将自定义配置文件加载进来。

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

**EnvironmentPostProcessor**接口允许用户在Spring Boot应用启动之前操作 `Environment`，[官方参考资料](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-customize-the-environment-or-application-context)。

[这个链接](https://www.baeldung.com/spring-boot-environmentpostprocessor)介绍如何载入和转换自定义的属性到  `Environment`中，并且最后再访问自定义的属性。

**可以用这个接口加载自定义yaml配置文件。**

#### 创建自定义配置文件

创建`person.yml`文件，内容很简单。

````yml
personconfig:
  name: lee
  age: 20
````

#### 实现EnvironmentPostProcessor接口，加载自定义的配置文件。

这段代码来自官方参考文档。

````java
public class PersonConfigProcessor  implements EnvironmentPostProcessor {

  private final YamlPropertySourceLoader loader = new YamlPropertySourceLoader();

  @Override
  public void postProcessEnvironment(ConfigurableEnvironment environment,
                                     SpringApplication application) {
    // 加载自定义的配置文件  
    Resource path = new ClassPathResource("persons.yml"); 
    PropertySource<?> propertySource = loadYaml(path);
    environment.getPropertySources().addLast(propertySource);

  }

  private PropertySource<?> loadYaml(Resource path) {
    if (!path.exists()) {
      throw new IllegalArgumentException("Resource " + path + " does not exist");
    }
    try {
      return this.loader.load("custom-person", path).get(0);
    }
    catch (IOException ex) {
      throw new IllegalStateException("Failed to load yaml configuration from " + path, ex);
    }
  }
}
````

#### 将接口的实现类注册

`PersonConfigProcessor` 必须注册到 `META-INF/spring.factories ` 文件中，目的是让Spring Boot在启动时能扫描到这个类，之后才能加载 。

````properties
org.springframework.boot.env.EnvironmentPostProcessor=com.yang.config.PersonConfigProcessor
````

#### 构建要绑定的对象

配置文件的目的就是为将其定义的属性注入到java object中，所以还需要构建一个类用来“接”配置文件定义的内容。

````java
@Component
@ConfigurationProperties(prefix = "personconfig")
@ToString
@Setter
public class PersonConfig {
  private String name;
  private int age;
}
````

这里的`@Setter` 不能省略，这种方式也是：先调用无参构造函数构造对象，再调用属性的set方法初始化。使用PersonConfig时，直接装配就可以了。

````java
@Autowired
private PersonConfig personConfig;
````

测试结果：

````java
PersonConfig(name=lee, age=20)
````



## 小结：

1. 如果不需要配置文件，直接注入，使用 `@Value`注解。
2. 用到了配置文件，但注入的都是单个属性值，将要注入的属性写在在 `application.properties/application.yml` 中，用`@Value ${}`方式注入。
3. 用到了配置文件，但注入的是个object，将要注入的object写在 `application.properties/application.yml` 中，用 `@ConfigurationProperties` 注入。
4. 自定义properties配置文件，选用`@PropertySource` 注入。
5. 自定义yaml配置文件，选用“实现EnvironmentPostProcessor”的方式注入。

其实，4和5的注入方式都是先将配置文件加载进来，再用@ConfigurationProperties绑定类。只是不同类型的文件，加载的方式不一样。

[github](https://github.com/diego1109/Spring-Boot-demo/tree/master/spring-boot-config)