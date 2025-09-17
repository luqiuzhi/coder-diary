# 深入理解Java中的Object类（JDK17版本）

## 一、Object的思想

**`Object` 类是 Java 中所有类的超类（根类）。** 这是一个非常核心的概念，可以从以下几个层面来理解：

1. **万物皆对象：** 在 Java 的类型体系中，除了八种基本数据类型（`byte`, `short`, `int`, `long`, `float`, `double`, `char`,
   `boolean`），所有引用类型都直接或间接继承自 `Object`。即使是数组（如 `String[]`）或者你自己定义的任何类，如果没有显式使用
   `extends` 继承另一个类，编译器都会自动让其继承 `Object`。

2. **多态的基石：** 正因为所有类都是 `Object` 的子类，所以一个 `Object` 类型的引用变量可以指向任何 Java 对象。
   ```java
   Object obj1 = "I am a String"; // String 是 Object 的子类
   Object obj2 = new ArrayList<>(); // ArrayList 是 Object 的子类
   Object obj3 = new MyCustomClass(); // 自定义类也是 Object 的子类
   ```
   这使得可以编写非常通用的代码，例如 `ArrayList`（内部用 `Object[]` 实现）在 JDK 5 之前可以存储任何类型的对象。

3. **提供通用行为契约：** `Object` 类定义了一组所有对象都具备的最基本行为的方法。这些方法构成了 Java
   对象模型的基石，其他类可以根据需要重写这些方法以提供特定的实现，但必须遵守这些方法定义的契约（Contract）。

4. **在 JDK 17 中的角色：** 这一点没有改变。即使是使用 `sealed` 类、`record` 类等新特性，`record` 类依然隐式继承自 `Object`。

---

## 二、理解 Object 类中的各个方法 (JDK 17)

我们逐一剖析 `Object` 类中的 9 个核心方法（不包括 `registerNatives`）。

### 1. `public final native Class<?> getClass()`

* **作用：** 返回此对象的运行时类（Runtime Class）。运行时类指的是对象在 JVM 中被实际创建时的类型，而不是引用变量的类型。
* **关键点：**
    * `final`： 此方法不能被重写，保证了所有对象获取 Class 对象的行为一致。
    * `native`： 由 JVM 本地实现，效率很高。
    * 返回值是 `Class<?>`，通配符 `?` 表示未知类型，强调了其通用性。
* **示例与用途：**
  ```java
  Object str = "Hello";
  Class<?> clazz = str.getClass(); // 获取的是 String.class
  System.out.println(clazz.getName()); // 输出 "java.lang.String"

  // 用途：反射（获取类信息、方法、字段等）
  if (clazz == String.class) {
      System.out.println("It's a String!");
  }
  ```

### 2. `public native int hashCode()`

* **作用：** 返回对象的哈希码值。主要用于支持哈希表（如 `HashMap`, `HashSet`, `Hashtable`）的高效运行。
* **契约（Contract）：**
    1. 在应用程序的一次执行过程中，对同一个对象多次调用 `hashCode()`，只要对象的 `equals` 比较所用的信息没有被修改，就必须返回相同的整数。
    2. 如果两个对象根据 `equals(Object)` 方法是相等的，那么调用它们的 `hashCode()` 必须返回相同的整数。
    3. **逆命题不一定成立：** 两个对象的 `hashCode()` 相同，它们并不一定 `equals`。但我们应该尽量避免这种情况（哈希冲突），以提高哈希表的性能。
* **关键点：**
    * 默认实现通常是将对象的内部地址转换成一个整数，但这并非强制性要求（如 OpenJDK 的默认实现可能与对象地址无关）。
    * **如果你重写了 `equals()` 方法，你必须重写 `hashCode()` 方法**，以确保契约的第 2 条成立。

### 3. `public boolean equals(Object obj)`

* **作用：** 指示其他某个对象是否与此对象“相等”。
* **契约：**
    1. **自反性：** `x.equals(x)` 必须返回 `true`。
    2. **对称性：** 如果 `x.equals(y)` 返回 `true`，那么 `y.equals(x)` 也必须返回 `true`。
    3. **传递性：** 如果 `x.equals(y)` 返回 `true`，且 `y.equals(z)` 返回 `true`，那么 `x.equals(z)` 也必须返回 `true`。
    4. **一致性：** 只要 `equals` 比较所用的信息没有被修改，多次调用 `x.equals(y)` 必须一致地返回 `true` 或一致地返回
       `false`。
    5. 对于任何非空引用 `x`，`x.equals(null)` 必须返回 `false`。
* **默认实现：** `(this == obj)`，即比较两个引用是否指向堆中的**同一个对象**（地址比较）。
* **关键点：**
    * 通常需要重写此方法来提供“逻辑相等”的比较（例如，比较两个 `String` 对象的内容是否相同）。
    * 重写 `equals` 时必须同时重写 `hashCode`。

### 4. `protected native Object clone() throws CloneNotSupportedException`

* **作用：** 创建并返回此对象的一个副本。“副本”的确切含义取决于该对象的类。
* **关键点：**
    * `protected`： 意味着如果一个类想要让外部代码调用其 `clone()` 方法，它必须重写此方法并将其可见性改为 `public`。
    * 要使用 `clone()`，类必须**实现 `Cloneable` 标记接口**。如果不实现，调用 `clone()` 会抛出
      `CloneNotSupportedException`。
    * 默认实现是**浅拷贝（Shallow Copy）**：它复制对象的所有字段。如果字段是基本类型，则复制其值；如果字段是引用类型，则复制其引用地址，而不是创建引用对象的新副本。
    * **深拷贝（Deep Copy）** 需要自己在重写的 `clone()` 方法中递归地克隆引用类型的字段。
    * 由于其复杂性和潜在问题（深拷贝/浅拷贝混淆、构造器未被调用），在现代 Java 开发中，**使用拷贝构造器或拷贝工厂方法是更受推荐的做法
      **。

### 5. `public String toString()`

* **作用：** 返回对象的字符串表示形式。该方法在对象被打印（如 `System.out.println(obj)`）或与字符串拼接时会被自动调用。
* **默认实现：** `getClass().getName() + "@" + Integer.toHexString(hashCode())`，例如 `java.lang.Object@1b6d3586`。
* **关键点：**
    * **强烈建议为所有自定义类重写此方法**，返回一个简洁但信息丰富、易于阅读的字符串，这对于调试和日志记录非常有帮助。
    * `record` 类会自动生成一个包含所有组件值的 `toString()` 方法。

### 6. `public final void wait() / wait(long timeout) / wait(long timeout, int nanos)`

* **作用：** 使当前线程等待，直到另一个线程调用该对象的 `notify()` 或 `notifyAll()` 方法，或者超过指定的等待时间。
* **关键点：**
    * `final`： 不能被重写。
    * 这些方法与线程同步密切相关，是 Java 固有锁（Intrinsic Lock / Monitor）机制的一部分。
    * **调用这些方法前，当前线程必须持有该对象的监视器锁（即必须在 `synchronized(obj)` 同步块内调用）**，否则会抛出
      `IllegalMonitorStateException`。
    * 调用 `wait()` 后，线程会释放该对象的锁，并进入等待状态。
    * 在现代并发编程中，更推荐使用 `java.util.concurrent` 包中的更高级的同步工具（如 `Lock`, `Condition`, `BlockingQueue`
      等），而不是直接使用 `wait()`/`notify()`。

### 7. `public final native void notify() / notifyAll()`

* **作用：** 唤醒在此对象监视器上等待的线程。`notify()` 随机唤醒一个，`notifyAll()` 唤醒所有。
* **关键点：**
    * `final` 和 `native`。
    * 同样，**调用线程必须持有该对象的监视器锁**。
    * `notifyAll()` 通常更常用，因为它能避免某些线程被永久遗忘的风险。

### 8. `protected void finalize()`

* **作用（已过时）：** 当垃圾收集器确定不存在对该对象的更多引用时，由对象的垃圾收集器调用此方法。原本设计用于在对象被回收前执行清理操作（如释放非堆内存资源）。
* **JDK 9 开始，此方法已被标记为 `deprecated`，并在 JDK 18 中进一步限制。**
* **强烈反对使用：**
    * 调用时机不确定，甚至不保证一定会被调用。
    * 性能开销大，会严重影响垃圾收集效率。
    * 可能导致资源泄漏、死锁等问题。
* **替代方案：** 使用 **try-with-resources** 语句或显式调用 `close()` 方法来管理资源（实现了 `AutoCloseable` 接口）。

---

## 总结与最佳实践

| 方法                | 重要性         | 是否常重写                | 说明与建议                                       |
|:------------------|:------------|:---------------------|:--------------------------------------------|
| `getClass()`      | 高           | 否 (`final`)          | 用于反射，获取运行时类型信息。                             |
| `hashCode()`      | 高           | **是** (若重写 `equals`) | 必须与 `equals()` 逻辑一致，遵守契约。                   |
| `equals()`        | 高           | **是**                | 实现逻辑相等比较，遵守五大契约。                            |
| `clone()`         | 中           | 选择性重写                | 谨慎使用。优先考虑拷贝构造器/工厂。                          |
| `toString()`      | 高           | **是**                | 返回有意义的描述，利于调试和日志。                           |
| `wait()/notify()` | 中           | 否 (`final`)          | 底层同步机制。优先使用 `java.util.concurrent`。         |
| `finalize()`      | **低 (已废弃)** | **绝对不要**             | 使用 `AutoCloseable` 和 try-with-resources 替代。 |

**核心思想：** `Object` 类定义了 Java 对象的“基本宪法”。理解它的每个方法，不仅仅是知道其功能，更重要的是理解其背后设计的**契约
**和**意图**。正确地重写这些方法（尤其是 `equals`, `hashCode`, `toString`）是编写健壮、正确、可维护的 Java 代码的关键。