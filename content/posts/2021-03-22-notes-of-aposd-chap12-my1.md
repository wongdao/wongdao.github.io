---
title: APoSD 阅读随想 - 代码维护的两种方法
slug: 2021-03-22-notes-of-aposd-chap12-my1
layout: post
date: 2021-03-22T09:50:00+08:00
categories: architecture
---
aPoSD：
> without documentation, future developers will have to rederive or guess at the developers original knowledge, this will take additional time, and there is a risk of buts if the new developer misunderstands the original designer's intentions.

<!--more-->

维护代码（new feature and bugfix）的方法有两种：1. 完全重写; 2. 打补丁

- 如果时间充裕，那么完全重写是最简单的办法。
- 如果是打补丁，那么前提是完全理解 code base，否则就不能确定新的补丁能如预期般工作。

这个和 diff patch 很相似：如果一个 patch 连 base 文件都没弄对，那么最终的 base + patch 就会有意料之外的结果。运气好的话问题在编译和测试时就能发现，不好的话就会被用户发现。
 
为了便于维护者（reader）正确地、能更高效地理解 code base，注释就是必不可少的。不管是重写还是打patch，好的注释都能提供代码缺失的隐式信息，这样能更好的帮助 reader 搞清原有业务逻辑，高效地写出代码。
