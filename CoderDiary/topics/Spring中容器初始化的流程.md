# Spring中容器初始化的流程

Spring Boot应用的启动入口是`SpringApplication.run(Class<?> primarySource, String... args)`方法。其核心启动流程可以概括为以下几个步骤：

1.  **实例化SpringApplication**： 解析启动类，推断应用类型（Servlet、Reactive等），初始化`ApplicationContextInitializer`和`ApplicationListener`。
2.  **运行SpringApplication** (`run`方法内部)： 这是最关键的步骤，它包含了我们下面要详细讲的主要阶段。

整个`run`方法的骨架清晰地展示了这个过程：

```java
public ConfigurableApplicationContext run(String... args) {
    // ... 一些初始化工作，如开始计时、配置Headless属性等

    // 1. 创建并启动监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        // 2. 准备环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        configureIgnoreBeanInfo(environment);

        // 3. 打印Banner
        Banner printedBanner = printBanner(environment);

        // 4. 创建应用上下文 (AnnotationConfigServletWebServerApplicationContext 等)
        context = createApplicationContext();

        // 5. 准备上下文（核心步骤之一）
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

        // 6. 刷新上下文（最最核心的步骤）
        refreshContext(context);

        // 7. 刷新后处理（默认为空实现）
        afterRefresh(context, applicationArguments);

        // ... 其他收尾工作：停止计时器、发布ApplicationStartedEvent等
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    return context;
}
```

---

## prepareContext 方法详解

在`createApplicationContext()`创建了空的`ApplicationContext`之后，`prepareContext`方法负责**为这个“空壳”上下文填充运行前所需的一切必要信息和配置**。你可以把它想象成新房子的“硬装修”阶段，布线、管道、墙面都已做好，但还没通电通水。

**主要工作内容包括：**

1.  **关联环境（Environment）**： 将之前准备好的`Environment`（包含配置属性、Profiles等）设置到`ApplicationContext`中。
2.  **执行上下文后处理**：
    *   调用所有的`ApplicationContextInitializer`。这些初始化器允许我们在上下文刷新（`refresh`）之前对其进一步配置或修改。
    *   发布`ApplicationContextInitializedEvent`事件，通知监听器上下文已准备好但尚未刷新。
3.  **向上下文添加BootstrapRegistry**： 将`BootstrapContext`中的单例对象添加到Spring容器中，这些通常是在非常早期阶段创建的组件（如`BootstrapConfiguration`中定义的Bean）。
4.  **设置BeanName生成器和资源加载器**。
5.  **最重要的：注册Bean定义（Bean Definitions）**：
    *   **注册主启动类**： 将`mainApplicationClass`作为一个Bean定义（`AnnotatedGenericBeanDefinition`）注册到容器中。这是整个应用的“配置源”，因为它上面有`@SpringBootApplication`注解，Spring会扫描这个类所在的包及其子包。
    *   **调用所有`BeanDefinitionLoader`**： 特别是，如果以“传统”方式启动（如War包），会加载XML配置等。但在现代Spring Boot应用中，这主要是为了确保主启动类被正确注册。
6.  **发布`ApplicationPreparedEvent`事件**： 通知所有监听器，上下文已准备就绪，Bean定义已加载，但尚未实例化任何Bean。

**总结：`prepareContext`的核心任务是“准备原材料”。它搭建好了Spring容器的框架，并将所有通过注解、配置等方式定义的Bean的“蓝图”（BeanDefinition）加载到了容器里，但还没有开始真正地创建（实例化）这些Bean。**

---

## refreshContext 方法详解

`refreshContext`方法是**整个Spring Boot（乃至Spring Framework）启动过程的灵魂和核心**。它直接调用了`ApplicationContext`的`refresh()`方法。这个方法负责根据`prepareContext`阶段准备好的“蓝图”，真正地**实例化Bean、依赖注入、初始化Bean，并启动内嵌的Web服务器**。继续用房子的比喻，这就是“通水通电、家具入户、正式入住”的阶段。

`refresh()`方法（在`AbstractApplicationContext`中定义）包含十几个关键步骤，以下是其最关键的几个：

1.  **准备刷新（prepareRefresh）**： 设置启动时间、活动状态，初始化属性源（PropertySources）占位符等。
2.  **获取并准备BeanFactory（obtainFreshBeanFactory）**： 对于GenericApplicationContext（Spring Boot所用），这一步只是返回已经加载好Bean定义的BeanFactory。如果是传统的XmlWebApplicationContext，这一步会负责解析XML文件并加载Bean定义。
3.  **准备BeanFactory（prepareBeanFactory）**： 配置BeanFactory的标准上下文特性，如类加载器、后置处理器（`ApplicationContextAwareProcessor`）、环境对象等。
4.  **后处理BeanFactory（postProcessBeanFactory）**： 允许子类在Bean定义加载完成后、Bean实例化之前，对BeanFactory进行进一步的修改。这是**Spring Boot扩展的关键点之一**，例如：
    *   `ServletWebServerApplicationContext`会在这里添加一个`WebServerFactoryCustomizerBeanPostProcessor`，用于后续配置内嵌Servlet容器。
5.  **调用BeanFactory后置处理器（invokeBeanFactoryPostProcessors）**： **这是极其重要的一步**。它会实例化并调用所有`BeanFactoryPostProcessor`。
    *   **`ConfigurationClassPostProcessor`**： 这个后置处理器会扫描所有带有`@Configuration`的类（包括我们的主启动类），解析`@ComponentScan`、`@Import`、`@Bean`等注解，从而找到**更多的Bean定义并注册到容器中**。这是Spring Boot自动配置的基石。
    *   **`PropertySourcesPlaceholderConfigurer` / `PropertySourcesPlaceholderConfigurer`**： 处理占位符`${...}`。
6.  **注册Bean后置处理器（registerBeanPostProcessors）**： 实例化并注册所有`BeanPostProcessor`（例如：`AutowiredAnnotationBeanPostProcessor`用于处理`@Autowired`注入）。这些处理器会在后续Bean的实例化过程中介入。
7.  **初始化消息源（initMessageSource）**： 国际化相关。
8.  **初始化事件多播器（initApplicationEventMulticaster）**： 用于应用事件（ApplicationEvent）的发布。
9.  **子类初始化（onRefresh）**： **这是Spring Boot启动内嵌Web服务器的关键钩子**！
    *   `ServletWebServerApplicationContext`会重写此方法。在这里，它会调用`ServletWebServerFactory`的`getWebServer()`方法**创建并启动内嵌的Tomcat、Jetty或Undertow服务器**。此时端口已经监听，但HTTP请求还不能被处理（因为DispatcherServlet等尚未初始化）。
10. **注册监听器（registerListeners）**： 将应用事件监听器注册到前面初始化的事件多播器中。
11. **完成BeanFactory初始化（finishBeanFactoryInitialization）**： **这是另一个最核心的步骤**。它实例化所有剩余的非懒加载单例Bean。
    *   这个过程包括：依赖注入（通过`BeanPostProcessor`）、调用初始化方法（如`@PostConstruct`、`InitializingBean`）等。
    *   **我们的业务Controller、Service、Repository等Bean都是在这个阶段被创建和组装的。**
12. **完成刷新（finishRefresh）**：
    *   发布`ContextRefreshedEvent`事件，表明容器已完全刷新。
    *   对于Web应用，还会初始化`LifecycleProcessor`并启动所有实现了`Lifecycle`的Bean。
13. **其他收尾工作**。

**总结：`refreshContext`（特别是其中的`refresh()`方法）是“施工和开业”阶段。它执行自动配置、解析所有配置类、实例化几乎所有Bean、解决依赖关系、执行初始化回调，并最终启动内嵌的Web服务器，让应用进入可服务状态。**

## 两者的关系与区别

| 特性       | `prepareContext`                  | `refreshContext` (核心是 `refresh()`) |
|:---------|:----------------------------------|:-----------------------------------|
| **核心任务** | **准备**： 搭建框架，加载蓝图（BeanDefinition） | **执行**： 根据蓝图施工，实例化并组装Bean          |
| **主要产出** | 一个充满了BeanDefinition的“空”容器         | 一个完全初始化、充满可用Bean、服务器已启动的“活”容器      |
| **关键事件** | `ApplicationPreparedEvent`        | `ContextRefreshedEvent`            |
| **比喻**   | 房屋的硬装、设计图、材料进场                    | 家具家电入户、通水通电、正式入住                   |

## 可以思考的问题

1. 在spring的容器初始化过程中，有哪些spring event？
2. spring是如何决定当前的容器是web容器还是非web容器？
3. spring是在什么时候处理spring.factories中的配置类的？在spring boot 3.x 版本之后，spring.factories已经被废弃，那么如何实现SPI机制？（这个问题可以单开一篇文章了）
