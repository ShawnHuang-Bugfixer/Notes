---
date: 2025-04-23
aliases:
  - Spring事务
---
# Spring 开启事务方式

Spring 提供了两种开启事务的方式：声明式事务和编程式事务。声明式事务使用注解 `@Transanctional`，编程式事务使用 `TransanctionManager` `TransanctionTemplate` 手动管理事务。

## 编程式事务

通过 `TransactionTemplate`或者`TransactionManager`手动管理事务。

```Java
@Autowired
private TransactionTemplate transactionTemplate;
public void testTransaction() {

        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {

                try {

                    // ....  业务代码
                } catch (Exception e){
                    //回滚
                    transactionStatus.setRollbackOnly();
                }

            }
        });
}
```

```Java
@Autowired
private PlatformTransactionManager transactionManager;

public void testTransaction() {

  TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
          try {
               // ....  业务代码
              transactionManager.commit(status);
          } catch (Exception e) {
              transactionManager.rollback(status);
          }
}
```

## 声明式事务

基于`@Transactional` 注解，底层使用了 `AOP` 对事务进行管理。该注解**只能应用到 public 方法上，否则不生效。**

>[[Spring-AOP]]

```Java
@Transactional(propagation = Propagation.REQUIRED)
public void aMethod {
  //do something
  B b = new B();
  C c = new C();
  b.bMethod();
  c.cMethod();
}
```

### @ Transactional 注解使用注意事项

- `@Transactional` 注解只有作用到 public 方法上事务才生效，不推荐在接口上使用；
- 避免同一个类中调用 `@Transactional` 注解的方法，这样会导致事务失效；
- 正确的设置 `@Transactional` 的 `rollbackFor` 和 `propagation` 属性，否则事务可能会回滚失败;
- 被 `@Transactional` 注解的方法所在的类必须被 Spring 管理，否则不生效；
- 底层使用的数据库必须支持事务机制，否则不生效；


# Spring 3 个重要的事务接口

Spring 框架中，事务管理相关最重要的 3 个接口如下：

- **`PlatformTransactionManager`**：（平台）事务管理器，Spring 事务策略的核心。
- **`TransactionDefinition`**：事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。
- **`TransactionStatus`**：事务运行状态。

## `PlatformTransactionManager` 平台事务管理器

`PlatformTransactionManager` 提供一套**统一的事务编程模型**，可以认为是 Spring 事务的 `SPI（Service Provider Interface）` 接口。

## `TransactionDefinition` 事务属性

定义了事务隔离级别、传播行为、超时、只读、回滚规则。

![[attachments/Pasted image 20250423111209.png]]

### 事务传播行为(propagation) 

事务传播行为定义了 **事务方法之间** 相互调用时，事务该如何传递。Spring 定义了多种事务传播行为:

---
**常用的事务传播行为**
1. **`TransactionDefinition.PROPAGATION_REQUIRED`**
	如果当前存在事务，则加入当前事务；没有则新建事务。  
	多个方法共享同一个事务，任何一个失败，整个事务回滚。
2. **`TransactionDefinition.PROPAGATION_REQUIRES_NEW`**
	每次调用都开启一个新事务，挂起当前事务。  
	各个事务互不影响，适用于日志记录、异步任务等子流程。
3. **`TransactionDefinition.PROPAGATION_NESTED`**
	 如果当前存在事务，则在当前事务中创建一个保存点，形成嵌套事务；没有事务时表现为新建事务。  
	嵌套事务回滚不会影响外部事务，但外部事务回滚将导致嵌套事务也回滚。
---
**不常用的事务传播行为**
4. **`TransactionDefinition.PROPAGATION_MANDATORY`**
5. **`TransactionDefinition.PROPAGATION_SUPPORTS`**
6. **`TransactionDefinition.PROPAGATION_NOT_SUPPORTED`**
7. **`TransactionDefinition.PROPAGATION_NEVER`**

### 事务超时属性

指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。

### 事务只读

执行多条 `DQL` 时，开启事务，保证 **读一致性**。开启事务的只读属性，数据库会进行优化。

### 事务回滚规则

事务回滚规则定义了哪些异常会导致事务回滚。事务只有遇到运行期异常（`RuntimeException` 的子类）时才会回滚，`Error` 也会导致事务回滚，但是，在遇到检查型（Checked）异常时不会回滚。

可以使用 `rollbackFor` 指定哪些异常触发事务回滚。

```Java
@Transactional(rollbackFor= MyException.class)
```