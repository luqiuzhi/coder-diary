# Redis

## Redis的数据结构类型

### Strings

#### 介绍

Redis的`strings`类型是最基础的数据类型，该类型是一串字节序列。 在`strings`类型中可以存储文本，序列化后的对象，以及二进制数组。

Redis中的`key`都是字符串类型，高效的`strings`结构是保证Redis如此高性能的一大原因。同时`strings`的查询复杂度是`O(1)`。

#### 常用命令

`strings`的使用非常简单，使用`GET`和`SET`命令即可。

```Bash
127.0.0.1:6379> set redis.learn.strings hello-rediss
OK
127.0.0.1:6379> get redis.learn.strings
"hello-rediss"
127.0.0.1:6379> set redis.learn.strings hello-world
OK
127.0.0.1:6379> get redis.learn.strings
"hello-world"
127.0.0.1:6379> 
```

使用Kotlin和`Jedis`客户端进行操作。

```Kotlin
fun main() {
    val jedisPool = JedisPool(JedisPoolConfig(), "localhost", 6379)

    try {
        val jedis = jedisPool.resource
        jedis.set("redis.learn.strings", "this is a value from kotlin!")
        val value = jedis.get("redis.learn.strings")
        println("value: $value")
    } catch (e: Exception) {
        e.printStackTrace()
    } finally {
        jedisPool.close()
    }
}
```

注意普通的`SET`命令在为一个已存在的key值**分配**值时，会覆盖原有的值。

针对默认的覆盖动作，Redis也提供了其他选项：仅在key存在时设置成功，或仅在key不存在时设置成功。

```Bash
127.0.0.1:6379> set redis.learn.strings hello nx
(nil)
127.0.0.1:6379> set redis.learn.strings hello xx
OK
127.0.0.1:6379> 
```

第一次设置`redis.learn.strings`返回失败，因为上面我们已经为这个key设置过值了。此时使用`NX`这个参数，表示`if not exist`。
第二次设置返回成功，此时使用了`XX`这个参数，表示`if exist`。

`strings`类型支持一次性设置多个键值对，使用`MSET`和`MGET`命令。

```Bash
127.0.0.1:6379> mset learn1 strings learn2 list learn3 set
OK
127.0.0.1:6379> mget learn1 learn2 learn3
1) "strings"
2) "list"
3) "set"
127.0.0.1:6379>
```

> Redis以前有一个`GETSET`命令，在设置新值的时候返回旧值。但是该命令在`6.2.0`版本中已经被标记为废弃，改为使用`SET`命令的参数来进行调用。
> 
> 这项改动其实跟`SETNX`,`SETEX`等命令的改动一样，都去掉了不同类型的`SET`命令，改为使用参数来表示不同的行为。在某种程度上也便于使用和记忆。

{style="warning"}

`strings`为数字类型的值提供了专门的原子操作：`INCR`、`DECR`、`INCRBY`、`DECRBY`。

```Bash
127.0.0.1:6379> set my_count 1
OK
127.0.0.1:6379> incr my_count
(integer) 2
127.0.0.1:6379> incr my_count
(integer) 3
127.0.0.1:6379> decr my_count
(integer) 2
127.0.0.1:6379> decr my_count
(integer) 1
127.0.0.1:6379> incrby my_count 3
(integer) 4
127.0.0.1:6379> decrby my_count 2
(integer) 2
127.0.0.1:6379> 
```

#### 应用场景

`strings`可以存储各种类型的字符串，比如使用该类型存储一张经过`BASE64`转码后的图片，或是存储一段`HTML`文本。

`strings`有以下几个常用的场景：

- 常规数据的缓存，如Token、网页文本、Session、序列化后的对象。
- 借助 `SETNX` 命令实现分布式锁。
- 可以用作计数器。Redis可以自动解析数字类型的`string`，并提供了支持原子操作的`INCR`、`DECR`、`INCRBY`、`DECRBY`。

***但是需要注意：`strings`的存储上限是512MB。***

### Lists

#### 介绍

Redis的`lists`是由双向链表进行实现的，链表中的值是字符串。

列表结构常常可以用来实现栈或是队列。

需要注意的是，基于双向链表实现的列表，具有`O(1)`的插入性能，但是访问性能并不如`Array`，因此如果需要快速访问，可以使用Redis中的`Sorted sets`数据结构。

`lists`的最大容量是2^32-1（4,294,967,295）。

#### 常用命令

针对列表类型，常用的`SET`和`GET`无法使用。有以下命令常用来操作列表：

- `LPUSH`：向列表头部添加一个元素。
- `RPUSH`：向列表尾部添加一个元素。
- `LPOP`：从头部移除一个元素，并返回这个元素。
- `RPOP`：从尾部移除一个元素，并返回这个元素。
- `LLEN`：返回列表的长度。
- `LMOVE`：将列表的单个元素移动到另一个列表，这是一个原子操作。
- `LRANGE`：返回列表中一段范围内的元素。
- `LTRIM`：截取列表。

对于列表，Redis提供了一些阻塞式命令：

- `BLPOP`：从头部移除一个元素，并返回这个元素，并且在列表中没有元素或命令未达到超时时间前一直阻塞该命令。
- `BLMOVE`：移动单个元素到另一个列表，并且在原列表没有元素时或命令未达到超时时间前一直阻塞该命令。

由于`lists`本身使用双向链表实现，在客户端中可以很轻松的将其视为栈（先进后出）或队列（先进先出）进行操作。

先进后出：

```Bash
127.0.0.1:6379> lpush redis.learn.lists one
(integer) 1
127.0.0.1:6379> lpush redis.learn.lists two
(integer) 2
127.0.0.1:6379> lpush redis.learn.lists three
(integer) 3
127.0.0.1:6379> lrange redis.learn.lists 0 -1
1) "three"
2) "two"
3) "one"
127.0.0.1:6379> lpop redis.learn.lists
"three"
127.0.0.1:6379> lrange redis.learn.lists 0 -1
1) "two"
2) "one"
127.0.0.1:6379> 
```

#### 应用场景

双向链表的实现让Redis的`lists`结构有很多适用场景，最常见的两个：

- 实现任务队列。可以阻塞式的弹出任务。
- 实现消息队列，和任务队列类似。但是使用`lists`结构实现的消息队列非常简陋，而`Redis 5.0`新增的`streams`数据结构较为完整的提供了消息队列的功能。
- 排行榜。但是只适合定时计算的排行榜，实时计算的可以使用`Sorted sets`。
- 最新消息。例如X的推文。

### Sets

#### 介绍

Redis中的`sets`表示无序集合，集合中的元素的唯一不重复的。

`sets`的最大容量同列表，也是2^32-1。

#### 常用命令

`sets`有以下常用命令：

- `SADD`：向集合中添加一个元素。
- `SREM`：从集合中移除一个元素。
- `SISMEMBER`：类似于`contains`的语义，判断一个元素是否存在于集合中。
- `SINTER`：交集操作，取多个集合中的交集，返回元素列表。
- `SCARD`：返回集合的元素个数。（a.k.a. cardinality）

（不得不说，Redis这几个数据结构的命令都很不一样。

#### 适用场景

集合本身的适用场景非常多，只要是无序唯一的场景都可以使用，比如：

- 参与抽奖的人数统计。
- 点赞列表。
- 还有微信的公众号的共同关注，可以使用差集计算得出。

### Hashes

#### 介绍

重量级选手登场——Hash结构！

虽然`strings`是最基础的结构，但是`hashes`可以说是最实用的结构，Hash结构最接近面向对象的数据结构，同时Hash可以对键值对进行“二级分组”，更方便数据的存取。

在`Redis 7.4`版本中为hash结构的单个属性增加了过期功能，让`hashes`的使用更加广泛。

理论上，每一个hash结构可以存储2^32-1个键值对，但是实际上能存储多少就得看实际的机器内存容量了。


### Sorted Sets