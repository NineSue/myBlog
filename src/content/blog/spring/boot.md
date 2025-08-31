---
title: '一道题瞅瞅Spring Boot启动流程'
author: '安'
description: '换种方式找jar包而已，之后可以学这种方式用在项目里。'
publishDate: '2025-08-31'
updatedDate: '2025-08-31'
tags:
- Spring
language: 'Chinese'
draft: false
heroImage: { src: './0.jpg', color: '#2F42C1' }
---
## 场景

在某 LOL 开发场景下，两组人开发了多个英雄及技能实现：

* **A 英雄**在主程序
* **B、C、D 英雄**在其他开发库（独立 jar）

如何 **不用 RPC**，利用 Spring 本身特性实现扫描到外部 jar 并注册 Bean？

---

## 答

* 外部 jar 的实现类上直接加 `@Component`（或者 `@Service`、`@Configuration` 都行）
* 同时在 jar 包里提供 `META-INF/spring.factories`，声明这个类作为 **自动配置类**

---

## 原理

整个流程涉及 Spring Boot 启动：

> Spring Boot 启动 → 创建 ApplicationContext → 扫描主工程 Bean → 扫描 spring.factories 中的外部配置类 → 注册 B、C、D Bean 到同一个容器 → 容器刷新完成

---

## 启动流程示意

```
┌──────────────────────────────┐
│ SpringApplication.run()       │
│ 启动 Spring Boot 应用        │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ ApplicationContext 创建       │
│ - AnnotationConfigApplicationContext │
│ - 容器初始化，管理所有 Bean   │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ 主程序扫描 Bean               │
│ - 扫描 @Component / @Service / @Configuration │
│ - ClassPathBeanDefinitionScanner               │
│ - 注册 A 英雄 Bean 到容器                      │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ 外部 jar 自动装配             │
│ ┌─────────────────────────┐ │
│ │ 1. spring.factories 扫描 │ │
│ │    - SpringFactoriesLoader │ │
│ │    - 加载外部配置类       │ │
│ │ 2. 配置类或带 @Component  │ │
│ │    - @Configuration / @Component │ │
│ │    - @Bean 方法实例化 Bean │ │
│ └─────────────────────────┘ │
│ - 注册 B / C / D Bean 到同一容器 │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ ApplicationContext 刷新完成   │
│ - 所有主程序 + 外部 Bean 完成初始化 │
│ - 生命周期管理由 Spring 控制 │
└─────────────┬────────────────┘
              │
              ▼
┌──────────────────────────────┐
│ 客户端注入使用               │
│ - @Autowired List<HeroUltimate> │
│ - 包含 A + B + C + D         │
│ - 统一调用英雄大招接口        │
└──────────────────────────────┘
```
