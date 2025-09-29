# 详解Java中的SPI机制

SPI，全称为Service Provider Interface，是 Java 语言的一个扩展机制。

SPI可以说是解耦思想的一种典型的实现方式。

## 一、组装电脑也是一种SPI {id="spi_1"}

想象一个场景：你要组装一台电脑。

1. **制定标准（接口）**：你定义好了电脑需要哪些部件，比如 `主板(Motherboard)`、`显卡(GraphicsCard)`、`内存(Memory)`。这些就是*
   *接口（Interface）**，它们规定了每个部件必须有什么功能（方法），但不关心具体是什么品牌和型号。
2. **市场采购（实现类）**：不同的厂商根据你的标准生产了自己的产品：
    * 华硕生产了 `AsusMotherboard`
    * 技嘉生产了 `GigabyteMotherboard`
    * 英伟达生产了 `NvidiaGraphicsCard`
    * 金士顿生产了 `KingstonMemory`
      这些就是**实现类（Implementation Classes）**。
3. **SPI 清单（配置文件）**：你不想把代码写死，非要某个特定品牌。你希望电脑城（Java
   运行环境）能根据一份“供应商清单”自动帮你找到所有符合标准的部件。于是，每个厂商都在一个固定的地方（`META-INF/services/`
   目录下）放了一个以接口**全限定名**命名的文件。
    * 文件 `META-INF/services/com.yourcompany.pc.Motherboard` 的内容是：
      ```
      com.yourcompany.pc.impl.AsusMotherboard
      com.yourcompany.pc.impl.GigabyteMotherboard
      ```
    * 文件 `META-INF/services/com.yourcompany.pc.GraphicsCard` 的内容是：
      ```
      com.yourcompany.pc.impl.NvidiaGraphicsCard
      ```
   这份“清单”就是 **SPI 的配置文件**。
4. **电脑城管理员（ServiceLoader）**：当你需要组装电脑，需要一个主板时，你不会自己new一个`AsusMotherboard`，而是会叫来电脑城的管理员
   `ServiceLoader`。
   ```java
   // 你对管理员说：给我找所有符合 Motherboard 标准的部件
   ServiceLoader<Motherboard> loader = ServiceLoader.load(Motherboard.class);
   // 管理员根据清单，把找到的所有品牌主板都拿给你
   for (Motherboard motherboard : loader) {
       // 你从里面挑一个（比如第一个）来用
       motherboard.install();
       break;
   }
   ```

这个管理员 `ServiceLoader` 的工作方式，就是 **SPI（Service Provider Interface）机制**。

它的核心思想是：**将接口的实现类的发现和加载解耦，不再由调用方硬编码指定，而是由第三方（服务提供者）声明，系统自动查找和装配。**

---

## 二、深入源码：看看 `ServiceLoader` 如何工作

我们打开 JDK 8 的 `ServiceLoader` 类（`java.util.ServiceLoader`），抓几个关键点来看。

> JDK 9+ 版本的 `ServiceLoader` 进行了大量的重构，并且增加了对模块化的支持，代码有较大变动，此处不做详细说明。
> 但是核心的服务发现机制仍然和旧版本一样。
> {style="note"}

### 1. 核心定义

```java
public final class ServiceLoader<S> implements Iterable<S> {
    // 配置文件所在的固定目录
    private static final String PREFIX = "META-INF/services/";

    // 正在加载的接口或抽象类的 Class 对象
    private final Class<S> service;

    // 用于加载服务实现的类加载器
    private final ClassLoader loader;

    // 内部缓存，存放已经加载并实例化好的服务对象
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 懒加载的迭代器，真正负责解析配置文件、加载类、实例化对象
    private LazyClassIterator lookupIterator;
    // ... 其他代码
}
```

从定义就能看出，它实现了 `Iterable` 接口，所以你可以用 `for-each` 循环来遍历所有服务实现。它内部有一个缓存 (`providers`)
和一個负责进行懒加载的迭代器 (`lookupIterator`)。

### 2. 入口方法：`load()`

当我们调用 `ServiceLoader.load(Motherboard.class)` 时：

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    // 使用当前线程的上下文类加载器 (Thread Context ClassLoader)
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return new ServiceLoader<>(service, cl);
}
```

它创建了一个新的 `ServiceLoader` 实例，并记录了要加载的接口 (`service`) 和使用的类加载器 (`cl`)。

### 3. 核心中的核心：迭代器 `Iterator` 与懒加载

`ServiceLoader` 是**懒加载**的。当你调用 `load()` 时，它并不会立即去读取所有配置文件并实例化所有类。真正的工作发生在你开始*
*遍历**它的时候。

当你调用 `loader.iterator()` 或使用 `for-each` 循环时，底层会用到那个 `LazyIterator`。

我们来看这个迭代器的 `hasNext()` 和 `next()` 方法做了啥（简化版逻辑）：

* **`hasNext()` / `next()`**: 当被调用时，迭代器会：
    1. **查找配置文件**：根据 `PREFIX + service.getName()` 的路径（例如 `META-INF/services/com.yourcompany.pc.Motherboard`
       ）找到所有配置文件。
    2. **解析文件内容**：读取文件中的每一行，每一行都是一个实现类的全限定名。
    3. **加载类**：使用 `ClassLoader.loadClass(line)` 来加载这个类。
    4. **实例化对象**：通过 `clazz.getConstructor().newInstance()` 反射创建该类的实例。
    5. **缓存对象**：将创建好的实例放入 `providers` 缓存中，下次再遍历时就直接从缓存取，避免重复加载和实例化。

**这就是为什么我们说 SPI 是“懒加载”的：只有当你真正遍历到某个实现时，它才会被加载和实例化。**

### 4. 如何使用：遍历服务

```java
// 1. 加载服务 (此时什么都没做，只是准备了加载器)
ServiceLoader<Motherboard> loader = ServiceLoader.load(Motherboard.class);

// 2. 遍历服务 (此时才开始真正干活！)
for (Motherboard motherboard : loader) { // 隐式调用了 iterator()
    // 在第一次进入循环时，迭代器开始解析文件、加载类、创建实例
    System.out.println(motherboard.getBrand()); // 输出: Asus
    // 通常我们会根据某种策略选择一个使用，比如第一个，或者基于内容的
    break;
}

// 3. 再次遍历会使用缓存，不会重新加载
for (Motherboard motherboard : loader) {
    System.out.println(motherboard.getBrand()); // 输出: Asus (来自缓存)
}
```

---

## 三、SPI 的优缺点（非常重要！）

### 优点：

* **解耦**：这是最大的优点。服务的提供者和调用者完全分离，调用者面向接口编程，不依赖任何具体的实现。实现类的变动完全不影响调用方。
* **可扩展性**：只要遵循接口约定，任何人都可以提供新的实现。只需打个 Jar 包，在 `META-INF/services/`
  下放个文件即可，无需修改原有代码。实现了“基于接口的插拔架构”。

### 缺点：

1. **一次性加载所有实现**：虽然懒加载，但 `ServiceLoader` 会加载配置文件中**所有**
   的实现类并实例化。如果有些实现初始化很耗资源，或者你只想用其中一个，这会造成浪费。
2. **不灵活**：它无法根据某种条件或参数来按需加载**某个特定**的实现。你只能通过遍历所有实现，然后自己用 `if` 判断来选择。
3. **线程安全问题**：`ServiceLoader` 本身的迭代器不是线程安全的。它的缓存 `providers` 也没有严格的同步机制，在多线程环境下需要自行处理。

---

## 四、JDK 17 有何不同？

在**核心机制上，JDK 17 的 SPI 与 JDK 8 甚至更早的版本相比没有任何变化**。`ServiceLoader` 的 API 和工作流程保持了高度的向后兼容性。

主要的差异来自于 JDK 9 引入的**模块化系统 (JPMS)**。在模块化项目中，SPI 的配置文件需要在新定义的 `module-info.java`
中声明，而不仅仅是依靠 `META-INF/services/` 目录。

**传统 Jar 包 vs 模块化 Jar 包：**

| 方式            | 如何提供 SPI 服务                                                                                                                     |
|:--------------|:--------------------------------------------------------------------------------------------------------------------------------|
| **传统 Jar 包**  | 只在 `META-INF/services/` 目录下提供配置文件。                                                                                              |
| **模块化 Jar 包** | 1. 在 `module-info.java` 中使用 `provides ... with ...` 语句声明。 <br/>2. (**可选**) 为了兼容非模块化环境，**依然可以**保留 `META-INF/services/` 目录下的配置文件。 |

**示例：一个模块如何声明自己提供了 `Motherboard` 接口的实现**

```java
// 在服务提供者的 module-info.java 中
module com.asus.pc.parts {
    requires com.yourcompany.pc.core; // 依赖定义了接口的模块
    provides com.yourcompany.pc.Motherboard // 声明我提供了这个服务
        with com.asus.pc.parts.impl.AsusMotherboard; // 具体的实现类是哪个
}
```

对于服务的**使用者**来说，代码完全不用变，还是 `ServiceLoader.load(...)`。`ServiceLoader`
在模块化环境下会优先从模块声明中获取服务提供者信息，如果找不到，还会回退到去 `META-INF/services/` 目录下查找，以保证兼容性。

## 简单回顾

* **SPI 是什么**：一种“服务发现”机制。接口是标准，实现类由第三方提供，系统自动查找和装配。
* **如何实现**：
    1. 定义接口。
    2. 提供实现类。
    3. 在 `META-INF/services/` 下创建以接口全限定名命名的文件，内容是实现类的全限定名（一行一个）。
    4. 通过 `ServiceLoader.load(接口类).iterator()` 来动态发现和加载所有实现。
* **核心类**：`ServiceLoader`，它通过懒加载迭代器的方式工作。
* **JDK 17**：核心机制不变，但增加了对模块化（`provides ... with ...` 语句）的支持。

这种机制在 Java 生态中应用极广，最经典的例子就是 JDBC 驱动加载 (`DriverManager`)、SLF4J 日志门面绑定、Spring Boot 的自动配置等。

## 常见的SPI框架对比 {id="spi_2"}

除了Java中原生的SPI机制，还有Spring SPI和Dubbo SPI。

| 特性         | Java SPI                   | Spring SPI (Spring Factories)			                                                                                               | Dubbo SPI                                                                         |
|:-----------|:---------------------------|:-------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------|
| **核心类**    | `java.util.ServiceLoader`  | `org.springframework.core.io.support.SpringFactoriesLoader`                                                                    | `org.apache.dubbo.common.extension.ExtensionLoader`                               |
| **配置文件位置** | `META-INF/services/接口全限定名` | `META-INF/spring.factories` (Boot 3.0+ 也支持 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`) | `META-INF/dubbo/internal/`, `META-INF/dubbo/`, `META-INF/services/` 等目录下的接口全限定名文件 |
| **配置文件格式** | 每行一个实现类全限定名                | 键值对（接口全限定名=实现类全限定名列表，逗号分隔）		                                                                                                   | 键值对（别名=实现类全限定名）                                                                   |
| **获取指定实现** | 不支持，需自行遍历判断                | 不支持，需自行遍历判断			                                                                                                                 | **支持**，通过 `getExtension("name")`                                                  |
| **按需加载**   | 否，一次性加载所有实现                | 否，一次性加载所有配置			                                                                                                                 | **是**，按名称加载时才实例化                                                                  |
| **依赖注入**   | 不支持                        | 与 Spring 容器无缝集成，支持完整的依赖注入和生命周期管理                                                                                               | **支持**，提供简单的 Setter 依赖注入                                                          |
| **AOP/包装** | 不支持                        | 支持 Spring AOP                                                                                                                  | **支持**，通过 Wrapper 机制（类似 AOP）                                                      |
| **自适应扩展**  | 不支持                        | 不支持                                                                                                                            | **支持**，通过 `@Adaptive` 和动态生成适配类                                                    |
| **自动激活**   | 不支持                        | 不支持                                                                                                                            | **支持**，通过 `@Activate` 按条件激活                                                       |
| **默认实现**   | 无明确约定                      | 无明确约定                                                                                                                          | **支持**，通过 `@SPI("defaultName")` 注解指定                                              |

### **Spring 的 SPI 机制**

Spring 的 SPI 机制主要通过 `SpringFactoriesLoader` 类实现，它在 Spring Boot 的自动配置中扮演了核心角色。

* **核心实现**：其核心逻辑是**扫描 classpath 下所有 `META-INF/spring.factories` 文件**，将其内容解析为键值对（Key
  为接口或抽象类的全限定名，Value 是以逗号分隔的实现类全限定名列表），并缓存起来。
* **与 Spring 容器的集成**：`SpringFactoriesLoader` 通常负责**读取配置并获取实现类的名称**。获取到这些类名后，Spring **容器**会负责实例化这些 Bean，并管理它们的完整生命周期（依赖注入、AOP 等）。这意味着 Spring SPI 的实现类可以享受 Spring 容器的一切特性。
* **主要应用场景**：主要用于 **Spring Boot 的自动配置**。Spring Boot 启动时会通过 `SpringFactoriesLoader` 加载
  `spring.factories` 中注册的大量自动配置类（如 `org.springframework.boot.autoconfigure.EnableAutoConfiguration`
  对应的配置类），从而自动配置应用程序上下文。

### **Dubbo 的 SPI 机制**

Dubbo 的 SPI 机制是其高扩展性的基石，功能非常丰富，由 `ExtensionLoader` 类实现。

* **核心实现与按需加载**：Dubbo SPI 的配置文件通常放置在 `META-INF/dubbo/` 或 `META-INF/dubbo/internal/`
  等目录下，文件名为接口的全限定名，文件内容为**键值对**（`name=implementationClass`）。`ExtensionLoader` 会**按需加载**
  扩展实现，只有在调用 `getExtension(name)` 时才会实例化对应的实现类，避免了资源浪费。
* **依赖注入 (IoC)**：Dubbo 提供了**简单的依赖注入功能**。如果一个扩展点有其他的扩展点作为依赖，Dubbo 会通过 **Setter 方法**自动注入所需的依赖实例。
* **自动包装 (AOP)**：Dubbo 支持**包装类** (Wrapper)。如果某个扩展实现类的构造函数**只有一个类型为接口的参数**，Dubbo
  会将其识别为包装类。当获取扩展点时，Dubbo 会**自动使用这些包装类层层包裹真正的实现实例**，从而实现类似 AOP
  的功能，用于在方法调用前后添加公共逻辑（如日志、监控等）。
* **自适应扩展点 (Adaptive)**：这是 Dubbo SPI 的一个高级特性。可以通过 `@Adaptive` 注解标记一个扩展接口的方法，甚至标记一个类。Dubbo
  会**动态生成适配类**，根据运行时参数（通常是 URL 中的参数）来决定实际调用哪个扩展实现。`getAdaptiveExtension()`
  方法用于获取自适应扩展点。
* **自动激活扩展点 (Activate)**：通过 `@Activate` 注解，可以根据给定的条件（如过滤器是作用于 Provider 还是 Consumer）**同时激活一组扩展实现**。这在 Dubbo 的 Filter 链等场景中非常有用。

### 三大机制的适用场景

* **Java SPI**：适用于**非常简单**的扩展场景，或者第三方库提供基本服务发现。
* **Spring SPI (`SpringFactoriesLoader`)**：是 **Spring Boot 自动配置的幕后英雄**。当你需要为 Spring Boot
  应用提供自动配置时，就应该使用它。
* **Dubbo SPI (`ExtensionLoader`)**：是 **Dubbo 框架高度可扩展性的核心**。功能最为丰富，提供了依赖注入、AOP、自适应扩展、自动激活等高级特性。

## 可以思考的问题

1. 为什么说Java原生的SPI机制不是线程安全的？多并发环境下会有什么样的后果？
2. spring boot 3.x 对比 2.x 中 SPI机制有何变化？
