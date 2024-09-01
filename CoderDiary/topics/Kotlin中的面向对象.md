# Kotlin中的面向对象

Kotlin也是一门面向对象的语言（毕竟是基于Java的）。 在Kotlin中声明一个类所用的关键字也是`class`。

那么Kotlin中的类和Java中的类有何不同？

## 类的声明

在类的声明上，有以下差异：

<tabs>
<tab title="Kotlin">

```Kotlin
class User(val id: Int, var name: String)
```

</tab>
<tab title="Java">

```Java
public class User {
    public Integer id;
    public String name;
}
```

</tab>
</tabs>

在Kotlin中，声明类的属性可以直接在`()`中间进行声明，除了声明属性的名称、类型之外，也可以直接设定默认值：

```Kotlin
class User(val id: Int, var name: String = "Bob")
```

在Kotlin中如果类只有属性没有方法，可以只按照上面的形式对类进行声明即可，不需要`{}`。 除此之外，属性也可以写在`{}`中：

```Kotlin
class User(val id: Int, var name: String = "Bob") {
    val email: String = "example@kotlin.com"
}
```

不得不说Kotlin的类声明比起Java来说写法要灵活不少（不过目前为止没啥大用）。

## 创建一个对象

在Kotlin中，创建对象不要`new`关键字。

```Kotlin
class User(val id: Int, var name: String)

fun main() {
    val user = User(1, "John")
}
```

直接使用类的构造函数来声明一个对象。

Kotlin中的构造函数与类同名，在默认情况下会提供一个和类声明具有同样入参的构造函数。

## 访问属性

访问属性使用`.`符号。

```Kotlin
class User(val id: Int, var name: String)

fun main() {
    val user = User(1, "John")
    println(user.id)
    println(user.name)
    user.name = "xxx@xx.com"
    println(user.name)
}
```

关于属性的访问控制下回再讲。

## 成员方法

成员方法需要生命在类的`{}`中。


```Kotlin
class User(val id: Int, var name: String) {
    fun printName() {
        println(name)
    }
}

fun main() {
    val user = User(1, "John")
    println(user.id)
    println(user.name)
    user.name = "xxx@xx.com"
    println(user.name)
    user.pringName()
}
```

方法调用和Java一样，通过对象进行调用。

## Data Class

Kotlin中有一种特殊的类——**数据类**，用以专门存储数据。

使用关键字`data class`进行声明：

```Kotlin
data class Log(val id: Int, val time: Long, val msg: String)
```

在Kotlin中为数据类默认提供了以下方法（有点类似使用Lombok注解的Java POJO类）：

| 方法               | 描述                  |
|------------------|---------------------|
| `toString()`     | 打印一个类的示例信息，包含可读取的属性 |
| `equals()`以及`==` | 实例的比较               |
| `copy()`         | 复制一个已有的对象来创建新的对象    |

不得不说Java如今也是紧跟潮流，在Java21中，提供了`Record`类来专门建立数据存储类。