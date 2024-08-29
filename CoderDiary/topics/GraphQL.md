# GraphQL

## 什么是GraphQL？

`GraphQL` 是一种新的API标准，并且该标准本身是建立 `REST` 类型的基础之上的。

借助官网对`GraphQL`的描述：

> At its core, GraphQL enables declarative data fetching where a client can specify exactly what data it needs from an API. 
> Instead of multiple endpoints that return fixed data structures, a GraphQL server only exposes a single endpoint and responds with precisely the data a client asked for.

我们可以简单总结`GraphQL`的几大特性：

1. ***声明式*** 的数据获取方式。
2. ***精准地*** 对请求进行定义。
3. 通过暴露 ***单一端点*** 提供服务，而不是由客户端自行组装。

## REST的尴尬处境

在`REST`类型的API中，资源以请求路径的形式进行暴露，往往每个路径只允许操作一类资源。

在WEB应用盛行的初期，`REST`标准几乎成为`http`接口编程的事实标准，但是在移动互联网迅速发展的当下，多样化的数据展示和跨平台数据请求的差异使得`REST`接口在请求服务器资源时显得有些力不从心。

### 移动互联网的盛行需要更加高效的数据请求方式

![网易云首页.jpg](网易云首页.jpg) { thumbnail="true" width="321" }

上面是网易云APP的首页，看起来界面比较 ~~复杂~~ 简洁，但其实首页锁请求的数据可不少。

首页大致可以分为以下区域：

- 顶部搜索视图
- Banner
- 圆形菜单按钮
- 推荐歌单
- 个性推荐
- 精选音乐视频
- 雷达歌单
- 热门播客
- 专属场景歌单
- 新歌，新碟，数字专辑
- 音乐日历
- 24小时播客
- 视频合辑
- ... 

每一个区域都有相应的资源获取接口，而开源社区也有人提供了简单易用的第三方API来获取网易云的数据: <a href="https://gitee.com/davie/NeteaseCloudMusicApi">NeteaseCloudMusicApi</a>

在这个开源框架中就是对网易云的各类资源进行请求并组合。（也算是GraphQL的低配实现吧）

### 跨平台下复杂的数据应用场景

如今众多应用不仅仅限于B/S架构，在跨平台开发大行其道的今天，往往是多个客户端对接一个服务端。

在不同客户端上资源的请求形式和内容可能各不相同，服务端难以用单一的接口形式满足不同平台的需求。

在阿里的`COLA`架构中，也需要针对不同的平台编写Adapter以适应不同平台的请求场景。

![cola-package.png](cola-package.png)

### 顺应快速迭代和敏捷开发的潮流

当下软件开发过程的另一趋势是 ***敏捷开发*** 。而**持续集成能力**作为其中的核心实践已经成为各大软件商的运维标准。

在产品快速开发和迭代的过程中，客户端的频繁改动是必然的。而在使用`REST`接口时，客户端的相应变更也会直接影响服务端对资源的暴露方式，而服务端的修改和发布往往是比较“费时费力”的。

## GraphQL解决了什么问题？ {id="graphql_1"}

假设以下场景（借用官网的例子，十分形象）：

![blog.png](blog.png)

图中是一个常见的博客网站首页，其中红色圈出来的三块区域，分别需要调取三个接口：**博主的用户信息** ；**博客列表** ；**关注列表**。

在博客首页渲染的过程中，往往会通过调用三个服务端接口并在前端组装的形式来呈现完整的页面：

![blog-app.png](blog-app.png)

而通过`GraphQL`可以实现以下效果：

![blog-grapg.png](blog-graph.png)

直观地来看，`GraphQL`位于客户端和服务端中间，相当于增加了一层数据处理。同时接口的请求数量也从三个下降至一个。

简单分析一下`GraphQL`的请求参数结构：

```Text
query {
    User(id: "er3tg439frjw") {
        name
        posts {
            title
        }
        followers(last:3) {
            name
        }
    }
}
```

`GraphQL`的参数结构并不是一个常规的`json`结构，而是自定义的语法——the GraphQL Schema Definition Language (SDL)。

其中`User`其实是在`GraphQL`中使用SDL定义的类型（`GraphQL`本身使用了一种强类型语义来定义API），以下是`User`的示例：

[//]: # (```graphql)

[//]: # (type User {)

[//]: # (    id: String!)

[//]: # (    name: String!)

[//]: # (    age: Int!)

[//]: # (})

[//]: # (```)

```Text
```
{ src="User.graphql" }

以下是Post的定义：

```Text
```
{ src="Post.graphql" }

其中`Post`可以保持对`User`的引用关系。