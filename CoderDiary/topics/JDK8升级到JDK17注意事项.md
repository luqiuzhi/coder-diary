# JDK8升级到JDK17注意事项

## 一、编译时问题（在代码编译和打包阶段）

1.  **最重要的变化：Jakarta EE 命名空间**
    *   **问题描述**：这是**最常见、最必然**会遇到的问题。从 Java EE 8 之后，Oracle 将 Java EE 移交给了 Eclipse 基金会，并重命名为 Jakarta EE。随之而来的最大变化就是所有 `javax.*` 的包名都被更名为 `jakarta.*`。
        *   `javax.servlet.*` -> `jakarta.servlet.*`
        *   `javax.persistence.*` -> `jakarta.persistence.*`
        *   `javax.annotation.*` -> `jakarta.annotation.*`
        *   `javax.websocket.*` -> `jakarta.websocket.*`
        *   `javax.faces.*` -> `jakarta.faces.*`
    *   **表现**：所有引用 `javax` 相关 API 的代码（如 HttpServlet, Filter, ServletContext 等）在 JDK 17 下编译时会出现“找不到符号”的错误。
    *   **解决方案**：
        *   升级应用服务器/ Servlet 容器到支持 Jakarta EE 的版本（如 Tomcat 10+, Jetty 11+, WildFly 26+ 等）。**注意：Tomcat 9 及之前仅支持 `javax`，Tomcat 10+ 支持 `jakarta`**。
        *   将所有依赖的框架也升级到兼容 Jakarta EE 的版本（见下文）。
        *   使用 **`jakarta.servlet:jakarta.servlet-api:5.0.0+`** 依赖替换旧的 `javax.servlet:javax.servlet-api:4.0.1`。

2.  **移除的内置 API（JAXB, JAX-WS, Corba 等）**
    *   **问题描述**：在 JDK 9 中，这些在 JDK 8 中内置的 Java EE 模块被标记为“ deprecated for removal”，并在 JDK 11 中被彻底移除。如果代码或依赖的库直接使用了它们（例如 `javax.xml.bind.JAXBContext`），编译会失败。
    *   **解决方案**：需要手动添加这些依赖到项目中（Maven/Gradle）。
        ```xml
        <!-- For JAXB -->
        <dependency>
            <groupId>jakarta.xml.bind</groupId>
            <artifactId>jakarta.xml.bind-api</artifactId>
            <version>3.0.1</version>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>3.0.1</version>
            <scope>runtime</scope>
        </dependency>

        <!-- For JAX-WS -->
        <dependency>
            <groupId>jakarta.xml.ws</groupId>
            <artifactId>jakarta.xml.ws-api</artifactId>
            <version>4.0.0</version>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.ws</groupId>
            <artifactId>jaxws-rt</artifactId>
            <version>3.0.0</version>
        </dependency>
        ```

3.  **访问控制（模块系统）**
    *   **问题描述**：JDK 9 引入了模块化系统（JPMS）。虽然大多数应用可以不创建模块描述符（`module-info.java`）而继续在“类路径”上运行，但一些深度依赖反射的库（如 Hibernate, Spring, 各种字节码操作库如 ASM, CGLib）可能会因为无法访问某些JDK内部API（如 `sun.misc.*`, `com.sun.*`）而失败。
    *   **表现**：抛出 `IllegalAccessError` 或 `InaccessibleObjectException`。
    *   **解决方案**：在启动命令中添加额外的 JVM 参数来开放这些模块。这是升级到 JDK 9+ 后非常常见的操作。
        ```bash
        # 开放所有模块的所有包（不推荐用于生产，但可用于快速验证）
        --add-opens=java.base/java.lang=ALL-UNNAMED
        # 更常见的做法是根据错误日志，按需开放特定模块
        --add-opens=java.base/sun.nio.ch=ALL-UNNAMED
        --add-opens=java.base/java.lang.reflect=ALL-UNNAMED
        --add-opens=java.base/java.io=ALL-UNNAMED
        --add-opens=java.base/java.util=ALL-UNNAMED
        ```

## 二、依赖和框架兼容性问题

第三方库和框架也必须与 JDK 17 兼容。

1.  **框架版本**
    *   **Spring**：需要 **Spring Framework 5.3+** (推荐 6.x) 和 **Spring Boot 2.7+** (推荐 3.x) 以获得对 JDK 17 的完整支持。**注意：Spring Boot 3.x 仅支持 Jakarta EE 9+**，这意味着它无法运行在 Tomcat 9 等旧容器上，并且Spring Boot 3.x 下已经废弃了spring.factories，自动装配的写法有所改变。
    *   **Hibernate**：需要 **Hibernate 5.6+** (推荐 6.x)。同样，新版本也切换到了 `jakarta.persistence`。
    *   **Jakarta Server Faces (JSF)**：需要 **Mojarra 4.0+**。
    *   **其他库**：检查像 Apache Commons, Log4j 2, SLF4J, Jackson, Gson 等常用库的最新版本，它们通常都兼容。

2.  **字节码操作库**
    *   **问题描述**：ASM, CGLib, Javassist 等库如果版本过旧，无法识别 JDK 17 的字节码格式，会导致动态代理、AOP等功能失败。
    *   **解决方案**：确保这些库升级到最新版本。通常，升级 Spring/Hibernate 时会间接解决这个问题，因为它们会带来新版本的字节码库。

3. **分布式组件升级**
   *    **熔断**：由网飞的hystrix组件，升级到spring-cloud-circuit-breaker
   *    **网关**：由zuul升级到spring-cloud-gateway
   *    **注册中心**：由eureka升级到nacos
   *    **链路追踪**：由sleuth切换到micrometer-tracing

## 三、运行时和行为变化

即使编译打包成功，应用在运行时也可能表现出不同。

1.  **废弃的 TLS 版本**
    *   **问题描述**：JDK 17 默认禁用了更老旧、不安全的 TLS 1.0 和 1.1 协议。如果应用需要与只支持这些旧协议的外部系统通信，会连接失败。
    *   **解决方案**：优先建议升级外部系统。如果不可行，可以通过 JVM 参数重新启用（**安全风险警告**）：`-Djdk.tls.client.protocols=TLSv1,TLSv1.1,TLSv1.2`。

2.  **默认垃圾收集器变化**
    *   **问题描述**：JDK 8 默认使用 Parallel GC (PS MarkSweep)，而 JDK 17 默认使用 G1GC。虽然 G1GC 对大多数场景更优，但可能因为应用特性不同而导致性能变化。
    *   **解决方案**：监控 GC 日志和应用性能。如果出现性能回退，可以尝试切换回旧的 GC（`-XX:+UseParallelGC`）或调试 G1GC 参数。

3.  **系统属性变化**
    *   **问题描述**：极少数系统属性可能被移除或改名。需要查阅官方发行说明。

4.  **URL 构造器行为**
    *   **问题描述**：从 JDK 某个版本开始，`new URL(...)` 的构造器行为有变，更推荐使用 `URI` 来构造然后再转换为 `URL`，以避免一些潜在的异常。
    *   **解决方案**：替换代码中的 `new URL(String)` 为 `URI.create(String).toURL()`。

## 四、工具链问题

1.  **IDE 配置**：确保 IntelliJ IDEA 或 Eclipse 版本支持 JDK 17，并正确配置项目的 SDK 和语言级别。
2.  **构建工具**：升级 Maven 到 3.6.3+，Gradle 到 7.x+，并确保相关的编译器插件（如 `maven-compiler-plugin`）版本支持 JDK 17（设置为 `3.8.0+`）。

## 五、老旧服务

1. **多JDK版本兼容**：某些服务或应用可能本身不支持JDK17，需要做好多个JDK版本同时使用的情况。

## 推荐升级步骤

1.  **准备阶段**：
    *   代码和依赖扫描：使用工具（如 `jdeps`）分析现有代码对已移除API的依赖。
    *   全面测试：确保在 JDK 8 下有一套通过的测试用例，作为基准。

2.  **增量升级**：
    *   **第一步：仅升级编译环境**。在 POM 中将 `maven-compiler-plugin` 的 `source` 和 `target` 设置为 `17`，尝试编译。此时会暴露大部分“移除的API”错误。
    *   **第二步：解决编译错误**。逐个解决上述问题，主要是添加替代依赖和替换 `javax` 为 `jakarta`。
    *   **第三步：升级依赖和框架**。将 Spring, Hibernate 等主要框架升级到兼容版本。**这是最复杂的一步**，可能需要大量代码调整。
    *   **第四步：解决运行时问题**。在测试环境中部署，根据错误日志添加 `--add-opens` 等 JVM 参数。
    *   **第五步：全面测试**：进行功能、性能、压力和安全测试。

3.  **部署上线**：在预生产环境充分验证后，再部署到生产环境。
