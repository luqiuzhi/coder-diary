# Spring注入bean的方式

Spring其实提供了多种注入bean的方式。

字段注入、构造器注入和setter注入是一种注入方式的划分，而注入的内容也有很多形式。

如最常见的单个bean的注入：

```Java
    @Autowired
    private BeanService oneServiceImpl;
```

当我们声明了一个接口时，也可以通过集合注入来同时注入多个同父类的bean：

```Java
    @Autowired
    private List<BeanService> serviceList;
```

同样的，也可以使用数组和Set来进行注入：

```Java
    @Autowired
    private BeanService[] services;
    
    @Autowired
    private Set<BeanService> serviceSet;
```

使用哈希结构来注入多个bean：

```Java
    @Autowired
    private Map<String, BeanService> serviceMap;
```

其中，key为bean的名称。

> 同父类多个bean的注入，常常用于hooks的场景，通过指定hook bean的顺序，还可以指定集合顺序，从而控制hook的执行顺序。

{style="tip"}

除了使用`@Autowired`进行注入，也可以使用构造器和setter方法。

当不确定bean是否存在时，还可以使用`Optional`进行注入，在注入阶段不会报空指针，使用时可以进行判空：

```Java
    @Autowired
    private Optional<BeanService> service;
    
    // use bean
    service.ifPresent(it -> {  // do something... })
```