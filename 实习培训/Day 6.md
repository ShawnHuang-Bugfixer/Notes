---
date: 2025-06-18
tags:
  - V8
  - UDC
---

致远学院深入学习 V8 客户端开发，包含以下课程：
- 实施方法论
- `V8` 平台-开发基础
- `V8` 平台培训-开发赋能

# 1. `V8` 平台-开发基础

## 1.1 `UDC` （unified design center） 统一应用设计中心

![[attachments/Pasted image 20250618134357.png]]

## 1.2 环境
![[attachments/Pasted image 20250618135809.png]]

## 1.3 后端代码实现自定义微流程

自定义微流程并发布 -> 拉取代码（阿里仓库）

![[attachments/Pasted image 20250618141744.png]]
### 1.3.1 后端项目结构
![[attachments/Pasted image 20250618140651.png]]
### 1.3.2 微流程后端日志追踪
直接下载运行日志或者直接查看
![[attachments/Pasted image 20250618143730.png]]

### 1.3.3 自定义微流程搭建细节

先定义自定义微流程，配置出入参，并为该微流程命名（所属接口禁止更改），最后构建应用。

从代码仓库拉取代码，构建规范的目录结构，最后进行编码。


![[attachments/Pasted image 20250618142739.png]]
![[attachments/Pasted image 20250618143042.png]]
![[attachments/Pasted image 20250618143109.png]]
# 2. `V8` 平台培训-开发赋能

`UDC` 代码模块构成分为四个部分：
- {应用名}{应用ID}-assemble
	启动类及其配置
- {应用名}{应用ID}-biz
	业务模块包含facade 模块接口实现类
- {应用名}{应用ID}-facade
	定义接口、 `DTO`和 `UDC` 定义的枚举
- {应用名}{应用ID}-web
	实体对应的 Controller 等等
## 2.1 `UDC` 应用后端拓展开发

主要分为三个场景：
- 自定义微流程接口
- 基于 Listener 监听实体增加、删除、修改时间
- 全链路代码实现，`controller` -> `appService` -> `service` -> `dao`

### 2.1.1 自定义微流程接口

自定义微流程后构建并拉取代码。

1. 在门面模块的自定义微流程包接口 `CustomMicroFlowAppService`  自动声明接口方法
	![[attachments/Pasted image 20250618152122.png]]
	![[attachments/Pasted image 20250618151621.png]]

2. 在业务模块实现 `CustomMicroFlowAppService` 接口中定义的方法。
	1）在模块下新建 extend 包
	2）实现业务逻辑
	![[attachments/Pasted image 20250618151940.png]]

3. push 并重新构建应用
	![[attachments/Pasted image 20250618152304.png]]
	发布后，只保留 extend 包，其它包代码会被重新覆盖。

### 2.1.2  无前端交互基于 Listener 监听实体

在 `{应用名}{应用ID}-biz` 模块下新建 `extend` 包，然后在该包下写事件监听处理逻辑。使用平台注解 `@EventHandler` 和 `@EventSubscriber`
![[attachments/Pasted image 20250618155753.png]]
同步监听模式下，触发实体的增删改查事件并执行对应监听方法时，共享事务。
![[attachments/Pasted image 20250618160047.png]]
### 2.1.3  