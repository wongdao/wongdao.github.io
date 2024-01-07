---
title: Data Binding vs. View Binding
date: 2020-03-26T14:00:00+08:00
slug: 2020-03-26-data-binding-vs-view-binding
type: posts
draft: false
categories:
  - android
tags:
  - MVVM
---

这两个不是互斥关系，可以同时使用。另外从功能上可以把 Data Binding 看作是 View Binding 的超集。

<!--more-->

# 原理

 - Data Binding: annotation，预编译时生成 binding class
 - View Binding: 猜测应该时 studio 或 gradle 生成 binding class (理由是视图绑定在 Android Studio 3.6 Canary 11 及更高版本中可用)

# 用途

 - Data Binding: MVVM
 - view binding: 替代 findviewbyid

# 对 layout 的要求

 - Data Binding: 需要 `<layout>` 开头
 - View Binding: 没有需求

# 数据方向

 - Data Binding: 双向（view to code + code to view）
 - View Binding: 只能是 code to view


# 标记语言

 - Data Binding: binding expressions
 - View Binding: 无

 
# 优缺点

 - Data Binding: 数据双向绑定，编译慢
 - View Binding: 单向绑定，编译快

 
* * *

参考:<br/>

- https://developer.android.com/topic/libraries/data-binding#java
- https://developer.android.com/topic/libraries/view-binding
