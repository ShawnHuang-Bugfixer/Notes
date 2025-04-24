---
date: 2025-04-24
tags:
  - 自动装配
---
`SpringBoot` 定义了一套规范，`SpringBoot` 程序启动时能够根据该规范自动地从实现了该规范的 Starter 中读取配置类并加载该配置类中的 Bean 到 `IoC` 容器。

# 自动装配流程

```mermaid
graph TD
    A[启动类标注 @SpringBootApplication] --> B[隐式启用 @EnableAutoConfiguration]
    B --> C[导入 AutoConfigurationImportSelector]
    C --> D[扫描 META-INF/spring.factories 和 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports]
    D --> E[加载所有配置类全限定名]
    E --> F[过滤条件: @ConditionalOnClass, @ConditionalOnMissingBean 等]
    F --> G[有效的自动配置类]
    G --> H[执行配置类中的 @Bean 方法]
    H --> I[Bean 注册到 IoC 容器]
    I --> J[完成自动装配]

    style A fill:#f9f,stroke:#333
    style J fill:#9f9,stroke:#090
```

# Spring Boot Stater

Spring Boot Starter 是依赖管理工具，将库和配置打包。

例如 `spring-boot-starter-web` 包含了构建 Web 应用程序需要的 Spring MVC、Tomcat（默认嵌入式服务器）、Jackson（JSON 库）等依赖，不需要我们手动添加和配置。​