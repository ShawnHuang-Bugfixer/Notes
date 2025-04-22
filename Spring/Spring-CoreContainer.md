---
date: 2025-04-22
aliases:
  - Spring核心模块
---
# Spring 核心容器

核心容器提供 `IoC` 依赖注入功能。

![[attachments/Pasted image 20250422145733.png]]

# `IoC` 控制反转与 `DI` 依赖注入

`IoC`（Inverse of Control:控制反转）是一种**设计思想**，通过 `IoC` 容器(Spring 框架) 来帮助我们实例化对象。`DI依赖注入` 是 `IoC` 的具体技术实现。

`IoC` 解决类之间的类之间依赖管理问题。

![[attachments/Pasted image 20250422161832.png]]

# Spring Bean

Bean 指的是被核心容器管理的对象，利用注解或者配置从核心容器中获取 Bean 并注入到所需位置。

通过 `XML` 配置或者注解方式，可以声明 `Bean` ，Spring 启动后，会加载 `Bean` 到核心容器中。

![[attachments/Pasted image 20250422150145.png]]
## Bean 的声明

Spring 中声明 Bean 有两种方式：XML配置声明和注解声明。

注解根据作用域又可以细分为作用于类和作用于方法两类：

作用于类：

- `@Component`：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
- `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- `@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 `Service` 层返回数据给前端页面。

作用于方法：

- `@Bean`：其修饰方法的所在类必须被 Spring 容器管理，推荐使用 `@Configuration` 注解类（Spring 会通过 `CGLIB` 代理增强该类，确保 `@Bean` 方法单例调用时返回同一实例。）

## Bean 的注入

注入 Bean 可以利用 `XML注入` 和 `注解注入` 两种方式。可以利用 `构造器` `setter` `字段` 进行 Bean 的注入。官方推荐使用 `构造器` 进行 Bean 注入。

常用的注解有两个 `@AutoWired` 和 `@Resource`，前者为 Spring 内置，后者为 `JDK` 内置。

`@AutoWired` 默认的注入方式为 `byType` 按照类型注入，当某类型有多个实例 Bean（使用 `@Quliafier` 指定 Bean 名称进行区分） 时，自动注入失败。

`@Resouce` 默认的注入方式为先 `byName` 按照 Bean 名字注入，后使用 `byType` 按照类型注入。推荐使用。


## Bean 的作用域

利用 `XML` 或者 @`Scope 注解` 可以指定 Bean 的作用域。 Spring 中 Bean 的作用域通常有下面几种：

- **singleton** : IoC 容器中只有唯一的 bean 实例。Spring 中的 bean 默认都是单例的，是对单例设计模式的应用。
- **prototype** : 每次获取都会创建一个新的 bean 实例。也就是说，连续 `getBean()` 两次，得到的是不同的 Bean 实例。
- **request** （仅 Web 应用可用）: 每一次 HTTP 请求都会产生一个新的 bean（请求 bean），该 bean 仅在当前 HTTP request 内有效。
- **session** （仅 Web 应用可用） : 每一次来自新 session 的 HTTP 请求都会产生一个新的 bean（会话 bean），该 bean 仅在当前 HTTP session 内有效。
- **application/global-session** （仅 Web 应用可用）：每个 Web 应用在启动时创建一个 Bean（应用 Bean），该 bean 仅在当前应用启动时间内有效。
- **websocket** （仅 Web 应用可用）：每一次 WebSocket 会话产生一个新的 bean。

## Bean 是否线程安全

线程是否安全取决于 Bean 实例的**状态**和**作用域**。作用域为 `proroType` 每次获取都创建新的 Bean 实例，相当于线程独有，不存在线程安全问题。作用域为 `singleton` 时，如果 Bean 中仅仅包含只读的共享资源，即 Bean 无状态，那么 Bean 为线程安全。

## Bean 的生命周期

Bean 的生命周期主要分为 4 个阶段：

1. 实例化
2. 属性赋值
3. 初始化
4. 销毁

![[attachments/Pasted image 20250422154158.png]]
### 1. ​**​实例化（Instantiation）​**​

- 通过构造器或工厂方法创建 Bean 实例
- ​**​注意​**​：此时对象属性均为默认值（未注入）

### 2. ​**​属性注入（Population）​**​

- 完成依赖注入（@Autowired、@Value 等）
- 实现 `BeanNameAware`/`BeanFactoryAware` 等接口的 Bean 会在此阶段后收到回调

---
**初始化 Bean 阶段**

### 3. **Aware 接口回调​**

**感知接口**，让 Bean 能够**感知（aware）到 Spring 容器**的特定对象或资源。通过实现这些接口，Bean 可以在初始化阶段获得 Spring 容器提供的各种基础设施支持。

Spring 容器主要分为两种类型：

1. ​**​`BeanFactory`​**​：基础容器，提供最基本的依赖注入功能

2. ​`ApplicationContext​​`：扩展自 `BeanFactory` 的高级容器

### 4. `BeanPostProcesser` 前置处理

通过实现 `BeanPostProcesser` 接口，可以自定义实现 Bean 的前置处理和后置处理。经过处理后，**Bean 会被修改**，**容器持有的是最终版本​**​是经过所有后处理器处理后的最终 Bean。

`SpringAOP` 就利用了 `BeanPostProcesser` 生成 Bean 的代理。

### 5.  `InitializingBean` 接口

Bean 实现接口后，重写 `afterPropertiesSet` 方法，在 Bean 初始化阶段被调用。强调 **面向接口**。

```Java
public class DatabaseService implements InitializingBean {
    private DataSource dataSource;
    
    @Override
    public void afterPropertiesSet() {
        // 确保dataSource已注入后才能建立连接
        this.connection = dataSource.getConnection();
        this.connection.validate(); // 验证连接有效性
    }
}
```

### 6. `init-method` 方法调用

初始化方法可以通过 XML/注解 两种方式进行配置，无需修改源代码。

```Java
@Configuration
public class AppConfig {
    @Bean(initMethod = "setup") // 指定初始化方法名
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// 普通 POJO（无需实现任何接口）
public class HikariDataSource {
    public void setup() { // 方法名自由定义
        System.out.println("连接池初始化...");
    }
}
```

### 7.  `BeanPostProcesser` 后置处理器

调用该接口中后置处理方法。

---
Bean 销毁阶段

### 8.  Bean 销毁阶段

容器关闭时进入 Bean 销毁阶段，此时容器会检查是否 Bean 是否实现 `Destruction` 接口，检查 Bean 是否定义了 `destry` 方法，根据情况调用。

