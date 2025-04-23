---
date: 2025-04-22
aliases:
  - 面向切面编程
---
# `AOP` 基本概念

`AOP`（Aspect Oriented Programming）即面向切面编程，`AOP` 是 `OOP`（面向对象编程）的一种延续，二者互补，并不对立。

`AOP` 把**公共的逻辑**（横切关注点）从业务逻辑从**抽离**，实现逻辑解耦。

![[attachments/Pasted image 20250422163456.png]]

# `AOP` 通知类型

- **Before**（前置通知）：目标对象的方法调用之前触发
- **After** （后置通知）：目标对象的方法调用之后触发
- **AfterReturning**（返回通知）：目标对象的方法调用完成，在返回结果值之后触发
- **AfterThrowing**（异常通知）：目标对象的方法运行中抛出 / 触发异常后触发。AfterReturning 和 AfterThrowing 两者互斥。如果方法调用成功无异常，则会有返回值；如果方法抛出了异常，则不会有返回值。
- **Around** （环绕通知）：编程式控制目标对象的方法调用。环绕通知是所有通知类型中可操作范围最大的一种，因为它可以直接拿到目标对象，以及要执行的方法，所以环绕通知可以任意的在目标对象的方法调用前后搞事，甚至不调用目标对象的方法

![[attachments/Pasted image 20250422163548.png]]

# AOP 切面执行顺序

切面类可以使用注解 `@Order` 或者实现接口 `Ordered` 来控制切面顺序。

# `SpringAOP` 底层实现

`Spring-AOP` 底层使用动态代理实现。被代理类如果实现了接口，则使用 `JDKProxy` 否则使用 `CGLibProxy` 。

`CGLibProxy` 通过**继承并重写方法**构造代理对象，要求类和方法不能被 `final` `private` `staic` 修饰。

`JDKProxy` 基于接口构造代理对象，接口本身不能为 `final`，接口中的方法默认都是 `public abstract`，不能为 `final`。

**`Spring AOP`（基于代理机制）仅能对 `public` 且 非 `final` 的方法生效，且必须通过代理对象调用才能生效。**
![[attachments/Pasted image 20250423105827.png]]
