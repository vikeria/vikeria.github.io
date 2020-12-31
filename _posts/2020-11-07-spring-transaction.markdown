---
layout:     post
title:      "Talk about spring transaction"
subtitle:   " \"Little share in group\""
date:       2020-11-07 23:26:00
author:     "vikeria"
header-img: "img/in-post/post-bg-spring-transaction.jpg"
tags:
    - Java
    - Spring
    - Transaction
---

> “Come on!”

## 前言
Spring框架大家一直都在使用，对于事物的使用大多数还是处于只是知道如何开启事物，使用@Transacional注解或者注入TransactionTemplate的方式去显示的使用。本篇就来谈一谈，spring是如何来管理数据库事务的。

## 正文

### 