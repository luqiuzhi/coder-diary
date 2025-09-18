# Spring中bean的初始化流程

Spring Bean 的生命周期本质上就是 **IoC 容器创建、组装、管理并最终销毁一个 Bean 对象的过程**。整个过程可以分为几个明确的阶段，其中许多阶段都提供了扩展点供开发者自定义。

---

## 1. 实例化 (Instantiation)
*   **Spring 的行为**：Spring 容器根据 Bean 的定义（配置元数据，如 XML、`@Bean`、`@Component` 等）来决定如何实例化一个 Bean。这通常通过**反射**调用其构造函数（默认是无参构造函数）来完成，但也可能通过工厂方法（Factory Method）或供应商（Supplier）等方式。
*   **源码级理解**：在 `AbstractAutowireCapableBeanFactory` 的 `createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)` 方法中执行。此时仅仅调用了构造函数，生成了一个“原始”对象，其属性均为默认值（如 null）。

## 2. 属性赋值 (Populating Properties / Dependency Injection)
*   **Spring 的行为**：容器将解析 Bean 所依赖的其他 Bean 和配置值，并通过反射（或 Method Handles）将其注入到目标属性中。这是 **“依赖注入” (DI)** 的核心体现。
*   **注入方式**：可以通过 Setter 方法、字段注入（`@Autowired`、`@Resource`）或构造函数注入（此时步骤 1 和 2 是合并的）。
*   **源码级理解**：在 `AbstractAutowireCapableBeanFactory` 的 `populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)` 方法中执行。该方法会处理 `@Autowired`、`@Value` 等注解。

## 3. Aware 接口回调 (Aware Interface Injection)
*   **Spring 的行为**：如果 Bean 实现了任何 Spring 的 `Aware` 接口，容器会在初始化阶段之前回调这些接口的方法，将相关的容器基础设施注入给 Bean 本身。
*   **常见的 Aware 接口**：
    *   `BeanNameAware`：设置 Bean 在容器中的 ID/Name。
    *   `BeanFactoryAware`：注入创建该 Bean 的 `BeanFactory` 实例。
    *   `ApplicationContextAware`：注入 `ApplicationContext` 实例（功能更强大，但会导致与 Spring 框架更强耦合）。
    *   `EnvironmentAware`, `ResourceLoaderAware` 等。
*   **源码级理解**：在 `AbstractAutowireCapableBeanFactory` 的 `initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd)` 方法中，通过 `invokeAwareMethods(...)` 和 `ApplicationContextAwareProcessor` 来处理。

## 4. BeanPostProcessor 前置处理 (BeanPostProcessor Pre-Processing)
*   **Spring 的行为**：这是 Spring 框架一个极其强大的扩展点。所有实现了 `BeanPostProcessor` 接口的 Bean 都会在容器中所有其他 Bean 的初始化阶段前后被调用。
*   **方法**：调用每个 `BeanPostProcessor` 的 `postProcessBeforeInitialization(Object bean, String beanName)` 方法。
*   **作用**：可以对 Bean 实例进行任何操作，例如包装、修改、甚至替换掉原始 Bean（通常返回一个代理对象）。**Spring AOP 正是基于此机制实现的**，它在后处理阶段为目标 Bean 创建代理对象。
*   **源码级理解**：在 `initializeBean` 方法中，调用 `applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)`。

## 5. 初始化 (Initialization)
*   **Spring 的行为**：执行 Bean 自定义的初始化逻辑。按顺序有以下三种方式：
    *   **`@PostConstruct` 注解**：执行被此注解标记的方法。（JSR-250 标准，推荐方式）
    *   **`InitializingBean` 接口**：如果 Bean 实现了此接口，调用其 `afterPropertiesSet()` 方法。
    *   **自定义 `init-method`**：执行在 Bean 定义中通过 `init-method` 属性（或 `@Bean(initMethod = "...")`）指定的方法。
*   **源码级理解**：在 `initializeBean` 方法中，依次触发上述回调。`@PostConstruct` 由 `CommonAnnotationBeanPostProcessor` 处理，`afterPropertiesSet` 和 `init-method` 在 `invokeInitMethods(...)` 中调用。

## 6. BeanPostProcessor 后置处理 (BeanPostProcessor Post-Processing)
*   **Spring 的行为**：再次调用所有 `BeanPostProcessor` 的 `postProcessAfterInitialization(Object bean, String beanName)` 方法。
*   **作用**：此时 Bean 已经完全初始化（属性已注入，自定义初始化方法已执行），可以进行最终的检查或包装。**AOP 的代理对象虽然在前置处理中可能已经开始创建，但通常是在这个后置处理步骤中最终返回给容器的。**
*   **源码级理解**：在 `initializeBean` 方法的最后，调用 `applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)`。

## 7. 完全可用 (Fully Initialized / In Use)
*   **Spring 的行为**：至此，Bean 已经完成了所有配置和初始化，被完全组装好，存放在 Spring 容器的单例缓存池（`singleton cache`）中。应用程序可以通过 `getBean()` 方法获取它并使用其服务。

## 8. 销毁 (Destruction)
*   **Spring 的行为**：当 Spring 容器（通常是 `ApplicationContext`）被关闭时，它会开始处理所有单例 Bean 的销毁逻辑，尤其是那些需要释放资源（如数据库连接、线程池）的 Bean。
*   **销毁方式的执行顺序**：
    *   **`@PreDestroy` 注解**：执行被此注解标记的方法。（JSR-250 标准，推荐方式）
    *   **`DisposableBean` 接口**：如果 Bean 实现了此接口，调用其 `destroy()` 方法。
    *   **自定义 `destroy-method`**：执行在 Bean 定义中通过 `destroy-method` 属性（或 `@Bean(destroyMethod = "...")`）指定的方法。
*   **源码级理解**：容器关闭时，会触发 `ConfigurableApplicationContext.close()`，最终在 `DefaultSingletonBeanRegistry` 的 `destroyBean(...)` 和 `destroySingletons()` 方法中处理销毁逻辑。`@PreDestroy` 由 `CommonAnnotationBeanPostProcessor` 处理。

---

