# Kotlin中的空指针安全

在Kotlin中，默认的变量都是非空的。比如声明一个字符串：

```Kotlin
fun main() {
    var s = "this is a string"
}
```

上面声明的`s`虽然使用了`var`关键字，也就是可修改变量，但是在后续的赋值中无法赋值为`null`。

下面的代码会直接编译报错：

```Kotlin
fun main() {
    var s = "this is a string"
    
    s = null
}
```

可以为空的变量该如何声明？使用`?`进行修饰。

```Kotlin
fun main() {
    var s: String? = "this is a string"
    
    s = null
}
```

`?`需要紧跟在变量的类型后面，当不能为空的变量被赋予空值时，编译器会报错：

```Text
Assignment type mismatch: actual type is 'kotlin.String?', but 'kotlin.String' was expected.
```

在Kotlin中`String`和`String?`是两种不同的类型。

## 访问可能为空的变量

当我们要访问可能为空的变量时，也需要借助`?`来进行访问。

```Kotlin
fun main() {
    var s: String? = "this is a string"
    
    s = null
    
    println(s?.length)
}
```

在上面的代码中，为了获取`s`的长度我们访问了`s`的**属性**：length。但是访问length时，需要使用`?.`而不是`.`。

如果直接使用`.`会产生以下编译错误：

```Text
Only safe (?.) or non-null asserted (!!.) calls are allowed on a nullable receiver of type 'kotlin.String?'.
```

可以看出使用`?`声明的变量在后续的使用时，都需要注意使用`?.`来访问其属性。这也在编译期对空指针的错误进行了预防。

## 使用Elvis运算符

首先不得不提一下这个运算符名称的由来，为啥叫Elvis？

`Elvis`其实指的是猫王（这里推荐一下奥斯汀·巴特勒主演的《猫王》这部电影，确实很好看），在wiki上有对这个名称由来的描述：

> The name "Elvis operator" refers to the fact that when its common notation, ?:, is viewed sideways, it resembles an emoticon of Elvis Presley with his signature hairstyle.

而我在另一个文章中找到了更加形象的描述（图源：<a href="https://itmob.cn/archives/why-called-elvis-operator">为什么 ?: 被称为猫王运算符（Elvis operator）</a>）：

![elvis.png](elvis.png)

![elvis-operator.png](elvis-operator.png)

说回正题，`?:`其实就是个二元运算符，仅仅针对左值为空的情况下，返回右值。

```Kotlin
fun main() {
    var s: String? = "this is a string"
    
    s = null
    
    println(s?.length ?: 0)
}
```

在上面的代码中，由于`s`为null，`s?.length`也为null。在左值为null的情况下，会返回右值，上面的代码最终会打印0作为结果。