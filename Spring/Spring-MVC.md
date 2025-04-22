---
date: 2025-04-22
aliases:
  - SpringMVC
---
# `SpringMVC` 设计思想

`MVC` 分为模型(Model)、视图(View)、控制器(Controller) ，通过将业务逻辑、数据、显示分离来组织代码。

Spring MVC 下我们一般把后端项目分为 Service 层（处理业务）、Dao 层（数据库操作）、Entity 层（实体类）、Controller 层(控制层，返回数据给前台页面)。

# `SpringMVC` 工作流程

## `SpringMVC` 核心组件

- **`DispatcherServlet`**：**核心的中央处理器**，负责接收请求、分发，并给予客户端响应。
- **`HandlerMapping`**：**处理器映射器**，根据 URL 去匹配查找能处理的 `Handler` ，并会将请求涉及到的拦截器和 `Handler` 一起封装。
- **`HandlerAdapter`**：**处理器适配器**，根据 `HandlerMapping` 找到的 `Handler` ，适配执行对应的 `Handler`；
- **`Handler`**：**请求处理器**，处理实际请求的处理器。
- **`ViewResolver`**：**视图解析器**，根据 `Handler` 返回的逻辑视图 / 视图，解析并渲染真正的视图，并传递给 `DispatcherServlet` 响应客户端

## `SpringMVC` 工作流程

传统开发模式返回视图，现在前后端分离模式下，只返回数据。使用注解 `@RestController`（组合了 `@Controller` 和 `@ResponseBody` ）实现。 

![[attachments/Pasted image 20250422165110.png]]
