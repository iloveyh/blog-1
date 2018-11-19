---
title: 微服务设计 - 版本控制
date: 2018-11-10
categories: [DevOps]
tags: [Version Control,Design]
---

# 背景
在微服务中，单个服务的升级改善是不可避免的，虽然改动最终引起Rest接口变动并不多，但仍然会出现。在《微服务设计》中，提供了两种处理方案：
1. 不同的接口共存
2. 同时运行多个版本服务

这本书的作者Sam Newman，他认为这两种方案都是可行的，但更倾向于不同接口共存。也提出了三点主要原因
- 当出现问题时，不同的接口共存可以更快的修改新老版本的代码并一同部署，而另一种就比较麻烦了
- 方案二维护困难，同时存在多个版本服务，对运维来说具有较大的挑战性
- 持久化层可能是一样的，方案二却有不同版本的实现，会导致潜在的复杂性

但我更倾向于方案二，该方案的实际使用者主要是Netflix（Spring Cloud便是基于他开源的微服务框架研发的），在现阶段（2018年）相对于作者的年代（2015年），技术上已经有了较大的变化。微服务的设计已经在很多公司大规模的推行及使用，变得更加成熟。作者所提出的前两点问题已经有了比较好的方案减少影响    

<!-- more -->

# 问题修复
首先，同时需要修改两服务的情况应当只有问题修复，新功能迭代，应发生在新服务上，老版本不应该支持（从某种程度上来说，可以启到督促用户升级）    

那么问题如何修复呢？同样也存在多种情况，我们拿书中的例子（用户服务）接着讲    

## 改进
用户服务负责的是用户相关的功能，其中一种情况是做了改进，增强部分功能。例如原本用户创建接口只是简单的用户名与单邮箱绑定（v1），改版后，需要实现的是多邮箱的绑定（v2），事情一个用户多个邮箱是比较常见的    

那么在这种情况下，底层的持久层已经改变了，原本应该用户表中存在邮箱字段，而新方案则为用户表与用户邮箱绑定表。我们需要在老节点拉出分支`release-v1`用于维护与兼容老版本，同时在`master`上迭代新的改进，稳定后拉出`release-v2`，不同的版本的服务与具体的分支对应，同样的可以创建`release-v3`等，但不建议同时维护多个版本，即便在不同的接口共存方案里也是一样的    

有了这些分支后，我们所做的修复都只需要在`master`上修复，通过`git cherry-pick`拉取单次提交至不同版本上，这是`GitLab Flow`方案，他所提倡的上游原则，在很大程度上避免了修改问题所造成的影响

## 重构
这里说的重构是指代码的重写，可能用了不同的技术架构或者语音，那么在这种情况下，不同的接口共存方案将无法处理，因为您的服务发生了彻底的变化

## 部署
问题修复后的部署也是难点，如果保证准确部署呢，CI/CD在很大程度上解决了这个问题，许多CI工具都提供指定分支的脚本配置，没错，我们可以提供一样的CI配置文件，保证编译测试一致的前提下，在不同的分支中运行不同的部署脚本

# 运行
运行时，如何将用户指向不同服务，同样的可以通过uri定位如`/v1/*`，微服务中网关与注册中心特别重要，我们可以在服务名中硬编码版本，比如`user-v1`这是第一个版本，在Spring Cloud中配置可以是：
```yml
spring:
  cloud:
    gateway:
      routes:
        - id: v1
          uri: lb://user-v1
          predicates:
            - Path=/v1/user/**
          filters:
            # Strip first path，such v1
            - StripPrefix=1
```
这并不困难

# 持久化
这个问题是唯一无法避免的，当然不同的接口共存也同样会遇到这个问题。作者认为旧接口转换处理后重新指向新接口服务，来避免持久化问题。这样的方案能解决的，在多服务中也很好解决，往往数据库中字段添加默认值就够了。其他无法兼容的，那么旧接口转换后访问新街口能不出错？唯一的措施是尽快推进升级    

# 其他
不同的接口共存就不存在问题么，显然不是的。每次新版本的发布，需要对所有接口重写，即使接口定义没有变化。这在很大程度上导致代码的冗余    

# 参考
- 《微服务设计》 Sam Newman 著 崔力强 张俊 译
- [GitLab Flow](https://docs.gitlab.com/ee/university/training/gitlab_flow.html)