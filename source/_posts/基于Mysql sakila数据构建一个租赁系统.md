---
title: "基于Mysql Sakila数据构建一个租赁系统"
catalog: true
date: 2020-07-28 22:51:24
subtitle: "学习笔记"
header-img: "/img/default.jpg"
tags:
- 后端
- Mysql
catagories:
---

Sakila是Mysql官网提供的Demo数据，业务是一个VCD出租业务，希望可以借此系统深入地学习一下Mysql。我一直奉行实践是学习最好方式，所以我希望能够基于这份数据构建一个完整的租赁系统，包括后台API和UI界面。

前端我打算使用React 16 + react-admin来构建基础框架，因为react-admin已经提供了完整的后台管理界面的layout，可以帮忙节省很多effert。UI 样式库为和react-admin保持统一而使用material-ui。

后端我打算使用sprint-boot + JPA + mysql来提供CRUD的APIs。

今天刚把后台搭建好，代码同步到Github上，慢慢来吧。
