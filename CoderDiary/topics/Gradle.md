# 初识Gradle以及踩坑记录

## 介绍

## 初次使用多模块

## Gradle Wrapper

`Gradle Wrapper`中定义了`Gradle`的版本，并对 **当前环境** 的Gradle发行版进行管理。

该部分的作用类似于前端环境中的`package-lock.json`，起到了版本控制的作用，不过`package-lock.json`仅仅对项目的依赖进行控制，不会管理使用的`Node.js`的版本。
而`Gradle Wrapper`管理了用于构建的`Gradle`的版本，保证各个开发机以及构建环境都使用了相同版本的`Gradle`。