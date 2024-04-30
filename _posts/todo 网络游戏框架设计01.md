---
title: "网络游戏框架设计|01 代码结构"
tags: 服务器
---

# 最基本的要求

- 热更新（客户端/服务的）
- 数据与逻辑方法（方便热重载等）
- 逻辑与表现分离（预表现，避免逻辑阻塞）
- ECS数据驱动

# ET版本演变

## ET 1.0

基础结构

- **Client**

  Main 主工程

  Hotfix 热更工程

- **Server**

  

## ET 2.0

对基础结构划分程序集，加快编译速度

- **Client**

  Core 框架核心库

  Loader 游戏侧脚本

  Hotfix 热更工程

  ThirdParty 第三方纯C#库

- **Server**

  Model 数据层

  Loader 游戏侧脚本

  Hotfix 热更工程

  ThirdParty 第三方纯C#库

  Hotfix 热更工程

## ET 3.0

客户端添加热重载

- **Client**

  Core 框架核心库

  Loader 游戏侧脚本

  Hotfix 热更工程

  Model 数据热更工程

  ThirdParty 第三方纯C#库

- **Server**

  没有变化

## ET 4.0

客户端表现层也需要热更，并拆分表现数据

- **Client**

  Core 框架核心库

  Loader 游戏侧脚本

  Hotfix 逻辑热更工程

  Model 逻辑数据热更工程

  HotfixView 表现热更工程

  ModelView 表现数据工程

  ThirdParty 第三方纯C#库

- **Server**

  没有变化