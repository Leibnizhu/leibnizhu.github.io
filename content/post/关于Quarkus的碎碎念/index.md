---
title: 关于Quarkus的碎碎念
date: 2022-05-15T16:03:10+08:00
draft: true
tags:
- Quarkus
- 云原生
image: thumbnail.jpg
---

最近把一个老项目迁到了Quarkus，想谈谈一些感想和想法。

## Why Quarkus???

答案很简单，for 云原生。在 [官网](https://quarkus.io/) 也可以看到Quarkus的宣传点也是围绕云原生进行的。  
云原生的实现大概有这些要点：

1. 应用/架构设计遵循 [12-Factor](https://12factor.net/)  
2. 服务容器化，最好框架层面就有支持
3. 启动速度快，便于伸缩。这里又包括镜像大小、layer设计、应用本身启动速度等因素。
4. 配合GitOps持续集成、持续交付