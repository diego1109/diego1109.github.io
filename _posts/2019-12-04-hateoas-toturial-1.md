---
layout: post
title: HATEOAS Link在Spring Boot中的使用（一）
date: 2019-12-04
categories: 学习笔记
tags: SpringBoot
comments: true
---

一直以来都想写一些东西，一是好记性不如烂笔头，把学过的东西整理出来加深映像，要是以后记忆模糊了可以翻过来看看；二是感觉写东西可以让我的心静下来，不会那么浮躁。这个link也是搞了好几次，学了忘了，忘了又学，正好今晚有时间把这个整理一下。

`HATEOAS(Hypertext as  the Engine of Application State)`解读字面意思就是：超文本作为应用状态的引擎，实则是客户端与服务器以一种新的交互方式：`链接`，而且这种交互的优点是客户端无需提前知道与服务器的交互规则是什么。不知道理解的对不对，大概解释一下：`RESTFUL API`中共有四个level，其中`level3`是我们最常用的，它通过不同的HTTP方法对资源进行不同的操作，并且使用HTTP状态码来表示不同的结果。比如：HTTP POST用来创建资源，返回201表示创建成功；HTTP GET表示获取资源，返回200表示获取成功。这种交互方式客户端必须知道资源的URL以及操作方式，如果后端发生改动而有没有及时更新文档，那前端的操作肯定会报异常了。HATEOAS属于`level4`它的好处是资源的URl是后端动态生成且主动的暴露给前端的（这里：前端和客户端是同一个意思，后端和服务器是同一个意思）,前后端是同步的，前端只需要通过”点击“的方式就能操作资源，就这么简单。还有一个优点是后端可以根据用户权限和资源状态动态的生成链接，这样连权限控制也直接给做了，总之就是HATEOAS好处多多。

Spring提供了`org.springframework.hateoas`依赖可以很快速的帮助开发者给资源创建URL，而其中主要使用到的是`ResourceSupport`、`Link`和`ControllLinkBuilder`这三个类。还有一个`Resource`类，它的源码描述是：**General helper to easily create a wrapper for a collection of entities，**其意思是轻松创建实体包装的通用助手，但是感觉它做出来的Link很丑，是真的很丑（第一次是听我老大说的，后来对比看了下，真是这样的，又丑又不好用）在最后面就会看到。

### 1、ResourceSupport

当每个资源需要为自己创建**resource representation**（我理解的是操作资源的各种链接，比如查找、删除、启动、停止等等）时都需要继承`ResourceSupport`基类，然后通过`add()`方法给自己添加需要的Link。

```java
@ToString
@Getter
@AllArgsConstructor
public class Customer extends ResourceSupport {
  private String customerName;
  private String customerId;
  private String companyName;
}

```

### 2、Link

`Link`中存储了资源的操作信息，比如我们创建一个静态的link表示一个customer的位置。

```java
Link link = new Link("http://localhost:8080/spring-security-rest/api/customers/10A");
```

Link类有两个成员变量很重要的，也很最常用，`rel`表示链接和资源的身份关系，在外层，`href`表示实际链接本身，在内层，显然两个是嵌套关系。

通过执行`customer.add(link);`将资源和他所需要的链接组合起来，得到的结果如下。`self`的意思很明显，这个链接表示的是资源本身所在的位置。前端只需要”点击“一下链接，就能获取到这个资源，跟HTTP GET有相同的效果，但在操作上确变的简单了好多。

```java
{
    "customerId": "10A",
    "customerName": "Jane",
    "customerCompany": "ABC Company",
    "_links":{
        "self":{
            "href":"http://localhost:8080/spring-security-rest/api/customers/10A"
         }
    }
}
```

### 3、ControllerLinkBuilder

上面的那条link是我们硬编码写的，如果不能实现动态生成Link那HATEOAS就失去了很大的优势了。`ControllerLinkBuilder`可以帮助我们动态生成Link。想一想对于那些没有resquestBody的HTTP操作，我们都是用特定的PathVariable拼出来从controller到method的URL对指定资源执行操作。而ControllerLinkBuilder的`linkTo()`方法就做了一件这样的事情，很简单，它拼出了需要的URL。生成方式也很简单：

```java
linkTo(CustomerController.class).slash(customer.getCustomerId()).withSelfRel();
```

其中：

- linkTo()找要生成的URL的个根映射（也就是基路径）在哪，一般都在Controller.class上。
- slash()在基路径后面追加子路径，这个方法是可以连续追加好几次的，直到拼出最后需要的URL为止。
- withSelfRel()他给URL添加了`self`和`href`两个属性。具体的意思和最后显示的格式在上面的介绍中都能找见。

这三个类的功能和主要的用法基本就是这样了。上面所有的链接存储的信息都是操作资源本身自己的，在有些复杂的场景中需要当前资源的链接是用来操作别的资源的。比如：客户customer和订单order之间是一对多关系，获取指定客户的全部订单这个需求是很常见的。

```java
@Getter
@ToString
public class Order extends ResourceSupport {
  private String orderId;
  private double price;
  private int quantity;
}

```

创建一个方法获取指定用户的所有订单：

```java
@GetMapping("/{customerId}/orders")
  public ResponseEntity getOrdersForCustomer(@PathVariable String customerId) {
    List<Order> orders = orderService.getAllOrdersForCuntomer(customerId);
    for (Order order : orders) {
      Link selfLink = linkTo(methodOn(CustomerController.class)
          .getOrderByIdForCustomer(customerId, order.getOrderId())).withSelfRel();
      order.add(selfLink);
    }
    Link link = linkTo(methodOn(CustomerController.class).getOrdersForCustomer(customerId))
        .withSelfRel();
    Resources<Order> result = new Resources<>(orders, link);
    return ResponseEntity.ok(result);
  }
```

这个套路也是很简单的，主要是执行下面这条语句**先从Controller上找出根路径，再用method追加上子路径，最后包装拼出的URL**。上面的方法做的事情也很简单：先给每一个`order`创建出自己的`selfLink`并组合，最后创建了一条`link`用来获取指定用户下的所有orders。`Resources`是HATEOAS提供的用来简化封装的类，它继承了`ResourceSupport`，只需要我们将resource和需要的link送给它，其他就不需要我们管了，自己完成了组合。

```java
Link link = linkTo(methodOn(CustomerController.class).getOrdersForCustomer(customerId))
        .withSelfRel();
```

最后看一下展示出来的效果，像_embedded、orderList这些多出来的字段能猜出来肯定是`Resources`这个泛型类给添加上去的。其实这个结果本身没有多少信息，但是这么多层嵌套和占用这么大的面积给人感觉好复杂的，不说唬住了小白起码也会让人觉得有点烦。可以考虑自己写resource类。

```java
{
    "_embedded": {
        "orderList": [
            {
                "orderId": "001A",
                "price": 250.0,
                "quantity": 25,
                "_links": {
                    "self": {
                        "href": "http://localhost/customers/10A/orders/001A"
                    }
                }
            },
            {
                "orderId": "002A",
                "price": 250.0,
                "quantity": 12,
                "_links": {
                    "self": {
                        "href": "http://localhost/customers/10A/orders/002A"
                    }
                }
            }
        ]
    },
    "_links": {
        "self": {
            "href": "http://localhost/customers/10A/orders"
        }
    }
}
```



写东西真的好费时间，这种感觉一直都没有变过。把脑海中的零碎片段梳理一下很多时候都要再查资料并且做实验的，不过好处是当这些都弄完的时候我觉着自己的认知更加清楚了些，这也许就是沉淀吧。骑着蜗牛继续向前跑~



参考文档：

[https://www.baeldung.com/spring-hateoas-tutorial](https://www.baeldung.com/spring-hateoas-tutorial)

