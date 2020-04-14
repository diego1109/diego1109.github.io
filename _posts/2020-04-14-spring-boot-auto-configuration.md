---

layout: post
title: spring boot 自动配置
date: 2020-04-14
categories: 学习笔记
tags: SpringBoot
comments: true 

---



Spring Boot内部对大量的第三方库或者Spring内部库进行了默认的配置，这些配置是否生效，取决于我们是否引入了对应库所需的依赖，而且通过全局配置文件可以对默认配置进行修改。自动配置的目的就是让第三方jar包里面的类很方便的添加到 `IOC`容器中去，大大简化了项目的配置过程。



## Spring Boot项目是怎么找到这些可以自动配置的第三方jar包中的类的？



```java
@SpringBootApplication
public class AutoconfigApplication {

  public static void main(String[] args) {
    SpringApplication.run(AutoconfigApplication.class, args);
  }

}
```

这是Spring Boot项目的启动入口，也叫主配置类。**@SpringBootApplication** 是个组合注解，内部如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

里面有一大堆注解，主要的就三个：

- `@SpringBootConfiguration`： 其实里面是个@Configuration注解，表明Spring Boot项目的入口类也是个配置类。
- `@ComponentScan`：被这个注解的类所在包以及子包下的所有”组件“都会被扫描，并将其bean添加到spring 容器里面。
-  `@EnableAutoConfiguration`：字面意思：启动自动配置，很显然用该是自动配置的入口了。进入~~

````java
Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
````

- `@AutoConfigurationPackage`：它的功能简单说就是将应用的root package给注册到Spring容器中，供后续使用。
- `@Import(AutoConfigurationImportSelector.class)`：字面意思：导入自动配置选择器。



> 看了好多博客，里面描述的都是@Import注解导入 `AutoConfigurationImportSelector`  后，调用了它里面的`selectImports()` 方法等等。我在这个方法里面打断点，之后启动debug，程序并没有在这个方法中停下来而是正常启动，所以应该是没有调用这个方法。我用的Spring Boot 2.2.6，但那么多人这么写，总不是空穴来风吧，这个之后在仔细研究吧。



但能确定的是自动配置调用了 `AutoConfigurationImportSelector` 类中的 `getAutoConfigurationEntry` 方法。

````java
protected AutoConfigurationEntry getAutoConfigurationEntry(
	AutoConfigurationMetadata autoConfigurationMetadata,
	AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取所有候选的配置。
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	// 将List放入Set再放入List，去掉重复的类名。
	configurations = removeDuplicates(configurations);
    // 获取不需要自动配置的类名。
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    // 检查需要移除的配置是否在候选列表中，如果有不存在的，会抛出异常。
	checkExcludedClasses(configurations, exclusions);
    // 从候选配置列表中删除不需要的配置。
	configurations.removeAll(exclusions);
    // 过滤掉不需要的配置。
	configurations = filter(configurations, autoConfigurationMetadata);
    // 将自动配置导入事件通知监听器。
    fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
````

获取类路径下所有jar包中的需要自动配置的类是通过 `getCandidateConfigurations()` 方法实现的。具体内容：

````java
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
	}
````

在这个方法中调用了 `SpringFactoriesLoader.loadFactoryNames` 方法去加载配置。最后以 `List<String>` 的形式将加载到的配置返回。

SpringFactoriesLoader.loadFactoryNames方法：里面也很简单，如下所示，里面定义了`factoryTypeName` 并且调用了 `loadSpringFactories(classLoader)` 方法后将结果返回，根据 `getOrDefault` 方法，我们应该能猜出`loadSpringFactories(classLoader)` 的返回值是个map类型的，再结果是以 `List<String>` 形式返回，就能确定`loadSpringFactories(classLoader)` 的返回值形式是：`Map<String, List<String>>`。（这段有点瞎扯，我分析这个干毛线啊，瞅一眼不不就行了吗）

````java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
	String factoryTypeName = factoryType.getName();
	return loadSpringFactories(classLoader).getOrDefault(factoryTypeName,Collections.emptyList());
	}
````

其实在Spring Boot项目中，可以自动配置的类的类名都存放在jar包中的`META-INF/spring.factories` 文件中，loadSpringFactories方法的功能就是从`META-INF/spring.factories`  文件中将所有可以自动配置的类的类名给读出来，并存放到`Map<String, List<String>>` 类型的对象中，最后返回，就干了这么一件事情。`getOrDefault(factoryTypeName, Collections.emptyList())` 的作用就是，从全部的可以自动配置的类中，找出我想要的那一组。`factoryTypeName` 的值就是spring.factories文件中等号左边的值，比如：ApplicationContextInitializer、ApplicationListener等等。

这个是spring-boot-autoconfigure-2.2.6.RELEASE.jar下的spring.factories文件内容：（当然别的jar包中也有spring.factories文件）。

````java
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition

.........
````

loadSpringFactories方法：

```java

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	// 先从缓存中加载这些可以自动配置的类名，如果能加载到就直接返回，如果不能再从文件中加载。
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null) {
		return result;
	}

	try {
        //拿到类路径下的所有jar包中的spring.factories的路径，并存放到urls变量中。
        // FACTORIES_RESOURCE_LOCATION = META-INF/spring.factories
		Enumeration<URL> urls = (classLoader != null ?
       classLoader.getResources(FACTORIES_RESOURCE_LOCATION) 	:ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		while (urls.hasMoreElements()) {
        // 拿出一个spring.factories的路径，扫描里面可以自动配置的类的类名。
		URL url = urls.nextElement();
		UrlResource resource = new UrlResource(url);
        /**
        * 将每一个spring.factories中的要自动跑配置的类名以key-value的形式封装成properties对象。
        * key就是等号左边的，value就是等号右边。
        */	
		Properties properties = PropertiesLoaderUtils.loadProperties(resource);
		for (Map.Entry<?, ?> entry : properties.entrySet()) {
			String factoryTypeName = ((String) entry.getKey()).trim();
			for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
            // 将可以自动配置的类的类名存放到result 中。
				result.add(factoryTypeName, factoryImplementationName.trim());
				}
			}
		}
        // 将可以自动配置的类的类名加载到缓存中。
		cache.put(classLoader, result);
		return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

在Spring Boot项目启动时可以自动配置的类就这样被全部扫描到了，接下来就是按照类的路径进行加载，创建bean并将其加入Spring容器中。



## 类找到了，自动配置是怎么实现的呢？

再来看下spring-boot-autoconfigure-2.2.6.RELEASE.jar下的spring.factories文件中 `EnableAutoConfiguration` 部分。下图中截取了其中一部分。

````java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudServiceConnectorsAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\

.........
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,
.........
````

可以看到，@EnableAutoConfiguration注解的自动配置类都是以 `XXXConfiguration` 结尾的，

我们以 `ServletWebServerFactoryAutoConfiguration` 为例看看里面到底是什么。

````java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
````

1. `@Configuration`：表明`ServletWebServerFactoryAutoConfiguration` 是个javaConfig类型的配置类，配置类的作用是给Spring 容器中添加bean的。
2. `@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)`： 表示这个类的配置顺序是高优先级。
3. `@ConditionalOnClass(ServletRequest.class)`：表示当ServletRequest.class在类路径下存在时，这个配置类才生效。
4. `@ConditionalOnWebApplication(type = Type.SERVLET)`：表示当前的Spring Boot应用程序是web应用程序的时候，这个配置类才生效。

5. `@EnableConfigurationProperties(ServerProperties.class)`： 这个注解比较重要了，自动配置就是从这开始的。EnableConfigurationProperties 字面意思：启动配置属性，它的实际作用是将`ServerProperties.class`生成bean，并添加到Spring 容器中。

6. `@Import()` :这个注解就是导入一些其他类的bean。

在这些注解中，3和4是条件注解，而且只有当这两个条件同事满足时，`ServletWebServerFactoryAutoConfiguration` 配置类才生效。

**那接下来还有个问题：@EnableConfigurationProperties将ServletRequest.class的bean添加到了Spring 容器中，那这个bean的属性的值是什么呢？**

先来看下 `ServerProperties.class` 类，也只截取了一小部分。

````java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {

	/**
	 * Server HTTP port.
	 */
	private Integer port;

	/**
	 * Network address to which the server should bind.
	 */
	private InetAddress address;
````

`@ConfigurationProperties`注解的功能是将全局配置文件中的属性和ServerProperties类中的属性绑定起来，说白了就是用配置文件中的属性更新ServerProperties类中对应的属性的默认值。`prefix = "server"`  指的是在全局配置中属性的前缀是什么。

在全局配置文件 `application.properties `中，我们常会看到一个配置：

````properties
server.port = 8080
````

这个配置是用来修改服务的HTTP端口的，当 `ServerProperties` 生成对应的bean时，就会拿 `server.port` 的值更新 `ServerProperties` 的 `port` 属性的默认值。

`ServerProperties` 每个属性都有[默认的值](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#server-properties)，当全局配置文件中没有配置对应属性时，`ServerProperties` 就会使用默认值来出生成bean。

## 总结

>
>
>ServletWebServerFactoryAutoConfiguration的自动配置过程总结一下：
>
>**目的：**
>
>自动配置ServletWebServerFactoryAutoConfiguration的目的是在Spring Boot项目启动时，将server相关的类生成对应的bean添加到Spring 容器中。
>
>**执行过程：**
>
>ServletWebServerFactoryAutoConfiguration类的路径在在spring.factories中被扫描到，在将添加server相关bean时，先是确定ServerProperties 中属性的值，通过@ConfigurationProperties注解从全局配置值中读取，如果能读到，则就用全局配置的中的值，如果读不到，就用默认值。@EnableConfigurationProperties注解将确定值的ServerProperties类生成bean并且添加到Spring 容器中去。接着是ServletWebServerFactoryAutoConfiguration类中的方法使用ServerProperties bean 去生成其他的 server bean，并添加到Spring 容器。这样跟server相关的组件在项目启动时就被加入到了spring容器中。
>
>EnableAutoConfiguration下的其他的XXXConfiguration的自动配置也是这样的流程。

