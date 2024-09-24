# Dubbo中的泛化调用

## 背景

在研究其他开源产品（以下简称A）的时候，发现了`Dubbo`框架提供的泛化调用。

A中提供了一个方法，作为函数调用的同一入口。通过这个方法调用函数，可以无视函数提供方的物理位置：函数可以是本地函数，也可以是远程服务。（是不是很像Actor模型？）

其中远程服务的调用使用到了`Dubbo`泛化调用的功能，于是去了解了一下。

## 定义

泛化调用分为客户端和服务端。

> 这是官网对客户端泛化调用的定义：泛化调用是指在调用方没有服务方提供的 API（SDK）的情况下，对服务方进行调用，并且可以正常拿到调用结果。
> 
> 这是官网对服务端泛化调用的定义：泛接口实现方式主要用于服务器端没有 API 接口及模型类元的情况，参数及返回值中的所有 POJO 均用 Map 表示，通常用于框架集成，比如：实现一个通用的远程服务 Mock 框架，可通过实现 GenericService 接口处理所有服务请求。

在官网的使用场景举例中提到了泛化调用对网关平台建设的作用：应用服务在接入网关后，客户端请求不会直接发给服务，而是**统一**发给网关，由网关调用服务。
此时网关不应知道服务提供的具体接口，而是使用一个通用的入口来统一调取服务，因此泛化调用可以作为这个**统一**入口。

之前一直使用`Zuul`和`Spring Cloud Gateway`网关，对于网关如何将请求给到服务一直没有深入了解，只当是使用Http进行请求的转发。

这个过程原理其实很简单，使用ip+port构造端点，利用客户端的请求构造`url`和`request payload`，向服务请求后再将`response`返回。

关于`Spring Cloud Gateway`的转发逻辑可以参考这篇文章：<a href="https://zhuanlan.zhihu.com/p/386866930">关于Spring Cloud Gateway与下游服务器的连接分析</a>，
从服务连接和接口转发的角度解释了源码。

在这个过程中本身是不需要涉及到`泛化调用`相关概念的，更不会存在网关和服务的接口依赖的问题。所以第一次看到Dubbo的这个问题，感到有些新奇。

而Dubbo之所有有`泛化调用`的概念，是源于设计理念的不同。

在Dubbo的服务通信中，本地方法和远程接口对于服务调用方来说都是`方法`，调用方不关心这个方法是否属于远程接口，使用什么协议，所以针对无法通过引用进行执行的代码，都通过`泛化调用`来执行。
而`泛化调用`的形式类似于Java的反射，通过invoke去执行具体的逻辑。

## 代码实现（参考官网）

注：该代码只是示例，无法直接执行。若需要可执行的例子，可以使用官方的example：<a href="https://github.com/apache/dubbo-samples/tree/master/2-advanced/dubbo-samples-generic">Dubbo的泛化调用</a>

在服务端暴露一个泛化方法，首先写一个包含实现逻辑的service，该service需要继承`GenericService`。

```Java
package com.foo;
public class MyGenericService implements GenericService {
 
    public Object $invoke(String methodName, String[] parameterTypes, Object[] args) throws GenericException {
        if ("sayHello".equals(methodName)) {
            return "Welcome " + args[0];
        }
    }
}
```

通过API的方式暴露这个方法。

```Java
// 用org.apache.dubbo.rpc.service.GenericService可以替代所有接口实现 
GenericService xxxService = new MyGenericService(); 

// 该实例很重量，里面封装了所有与注册中心及服务提供方连接，请缓存 
ServiceConfig<GenericService> service = new ServiceConfig<GenericService>();
// 弱类型接口名 
service.setInterface("com.xxx.XxxService");  
service.setVersion("1.0.0"); 
// 指向一个通用服务实现 
service.setRef(xxxService); 
 
// 暴露及注册服务 
service.export();
```

上面的代码中，`service.setInterface("com.xxx.XxxService")`所指定的接口名可以自定义，保证不重复就好了，这也是`弱类型`的特点。
因为实际是不存在这个接口的，甚至连包路径可能都不存在。

客户端调用的逻辑：

```Java
//创建ApplicationConfig
ApplicationConfig applicationConfig = new ApplicationConfig();
applicationConfig.setName("generic-call-consumer");
//创建注册中心配置
RegistryConfig registryConfig = new RegistryConfig();
registryConfig.setAddress("zookeeper://127.0.0.1:2181");
//创建服务引用配置
ReferenceConfig<GenericService> referenceConfig = new ReferenceConfig<>();
//设置接口
referenceConfig.setInterface("org.apache.dubbo.samples.generic.call.api.HelloService");
applicationConfig.setRegistry(registryConfig);
referenceConfig.setApplication(applicationConfig);
//重点：设置为泛化调用
//注：不再推荐使用参数为布尔值的setGeneric函数
//应该使用referenceConfig.setGeneric("true")代替
referenceConfig.setGeneric(true);
//设置异步，不必须，根据业务而定。
referenceConfig.setAsync(true);
//设置超时时间
referenceConfig.setTimeout(7000);

//获取服务，由于是泛化调用，所以获取的一定是GenericService类型
genericService = referenceConfig.get();

//使用GenericService类对象的$invoke方法可以代替原方法使用
//第一个参数是需要调用的方法名
//第二个参数是需要调用的方法的参数类型数组，为String数组，里面存入参数的全类名。
//第三个参数是需要调用的方法的参数数组，为Object数组，里面存入需要的参数。
Object result = genericService.$invoke("sayHello", new String[]{"java.lang.String"}, new Object[]{"world"});
```

其中的关键是最后两行：

```Java
genericService = referenceConfig.get();
        
Object result = genericService.$invoke("sayHello", new String[]{"java.lang.String"}, new Object[]{"world"});
```

根据相同的`interface`和`version`构造出`genericService`后，就可以直接调用其`$invoke`方法。
通过指定方法名、参数列表的信息，即可调用成功。

在泛化调用的情况下，调用方可以不依赖服务方的任何一行代码，对暴露出的远程方法进行调用。

> 总结：泛化调用的本质其实也是通过代理在消费方生成不同的执行逻辑，在代理中会执行具体的远程方法的调用，其实也是通过RPC的方式去调用。
> 这在行为上和`FeignClient`其实是一致的，只是后者使用了声明式客户端，而前者使用`泛化`的概念来进行调用。