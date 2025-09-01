Spring State Machine 是一个通用的状态机框架，用于描述业务实体在不同状态之间的有序、可控切换过程。核心思想是用事件驱动的有向图（状态→事件→状态）代替硬编码的 if/else 或 switch 流程，让状态逻辑更清晰、更容易维护和扩展。

# Spring State Machine 核心组件
Spring State Machine 仅仅定义了一套通用的状态机框架，使用该框架时，首先需要绘制业务状态机，厘清有哪些状态，哪些事件会触发状态流转以及状态流转过程中需要执行的动作。现在从概念入手，掌握 Spring Sate Machine 的核心组件：
- **状态(State)**
	枚举，涵盖业务状态机中所有涉及到的状态。
- **事件(Event)**
	枚举，涵盖业务状态机触发状态流转的事件。
- **状态转换(Transition)**
	定义源状态到目标状态的转换，并关联事件等内容。
- **守卫条件(Guard)**
	Boolean 判断逻辑，控制某次状态转换是否可以执行。
- **动作(Action)**
	定义状态流转过程中执行的一系列动作，比如持久化状态等操作。
- **拓展状态(ExtendedState)**
	存储临时键值对，供其他组件读写，支持跨状态传递数据。
- **状态机实例(StateMachine)**
	状态机实例，利用该实例触发状态流程。
# Spring State Machine 核心工作流程

Spring State Machine 的核心思想即事件触发状态流转，现围绕该思想描述其运行时工作流程：
- **定义状态与事件**
- **配置状态机**
    主要配置状态转换以及转换守卫条件和执行动作。
- **获取状态机实例**
	使用 `StateMachineFactory` 创建实例，必须调用 `stateMachine.start()` 启动状态机。若需要从特定状态开始，可用 `StateMachineAccessor` 重置状态。
- **触发事件驱动转换**
	通过 `stateMachine.sendEvent(Message<Event>)` 发送事件，事件可携带 payload 和 headers。
	状态机根据当前状态与事件，找到匹配的 transition：
	1. 先执行 Guard 判断是否允许；
	2. 若允许：按顺序执行 exit action → transition action → entry action（外部转换），或只执行 transition action（内部转换）；
	3. 更新当前状态。
- **维护上下文和变量**
	状态机内部的 **ExtendedState** 是一个键值存储区，可在 Guard/Action 中共享数据。
	事件 Message 的 headers 也可携带临时数据。
- **监听与持久化（可选）**
	可通过 `StateMachineListener` 监听状态变化、事件未处理、异常等。
	若要在分布式或重启后恢复状态，可使用 `StateMachinePersister` 保存与加载 `StateMachineContext`。
- **关闭或销毁状态机（可选）**
    使用 `stateMachine.stop()` 正常关闭状态机。

# Spring State Machine 使用案例

以图片审核过程为例，初始版本的图片审核极为简陋，图片状态仅包含未过审和已过审两种，且图片是否过审需要人工评判，当上传图片数量逐步增加后，人工审核效率极为低下且容易出错。为了优化该问题，参考了 Bilibili 稿件发布审核流程，在 ChatGPT 的辅助下，设计了全新的图片审核流程。

## 图片审核状态机
以审核方式划分可以分为**机审** -> **复审** -> **申诉** 三种，以审核结果划分可以粗略划分为 **通过**、**未通过** 和 **存疑** 三种。图片审核状态机如下：

| 审核结果 \ 审核方式 | 机审            | 复审              | 申诉              |
| ----------- | ------------- | --------------- | --------------- |
| **通过**      | AI_PASS       | MANUAL_PASS     | APPEAL_PASS     |
| **未通过**     | AI_REJECTED   | MANUAL_REJECTED | APPEAL_REJECTED |
| **存疑**      | AI_SUSPICIOUS | _无此状态_          | _无此状态_          |
同时为了简化其他服务的判断，设计了 `FINAL_APPROVED` 和 `FINAL_REJECTED` 最终状态，这也符合**开闭原则**，当需要添加其他状态时，不必修改其他的代码。

## AI 机审
引入腾讯云数据万象第三方服务进行对图片进行机审。接口文档明确说明调用审核接口等待审核结果有异步响应和同步响应两种方式。异步响应通过调用我方系统暴露出的接口传回响应结果，这种方式不利用在开发时使用，因为开发环境定义的接口无法暴露给三方服务。为了便于测试，故采用同步调用方式。

经测试，同步调用等待审核结果的响应时长根据图片质量而不同，平均耗时在 500 ms 上下。如果不解耦图片上传和图片审核，会导致用户上传图片体验变差，当高并发上传的情况下，由于线程被阻塞，很可能导致服务不可用甚至崩溃。故使用消息队列解耦图片上传和图片审核。

## Spring State Machine 配置代码演示
每张图片独立维护一个状态机实例，避免全局同步开销。通过 `StateMachineFactory` 快速创建线程安全实例，重置初始状态、启动状态机，驱动事件完成状态流转，并在 Action 或 Listener 中持久化状态。Guard 用于拦截非法状态流转，Action 执行业务逻辑，确保状态流转清晰且可扩展。

1. 事件
```Java
public enum ImageReviewEvent {  
    AI_REVIEW_PASS,  
    AI_REVIEW_REJECT,  
    AI_REVIEW_SUSPICIOUS,  
    MANUAL_REVIEW_PASS,  
    MANUAL_REVIEW_REJECT,  
    APPEAL_SUBMIT,  
    APPEAL_PASS,  
    APPEAL_REJECT,  
}
```
2. 状态
```Java 
public enum ImageReviewState {  
    PENDING_REVIEW,  
    AI_PASS,  
    AI_REJECTED,  
    AI_SUSPICIOUS,  
    MANUAL_PASS,  
    MANUAL_REJECTED,  
    FINAL_APPROVED,  
    FINAL_REJECTED,  
    APPEAL_PENDING,  
    APPEAL_PASS,  
    APPEAL_REJECTED  
}
```
3. 守卫
```Java
public Guard<ImageReviewState, ImageReviewEvent> AppealGuard() {  
    return stateContext -> Boolean.TRUE.equals(transactionTemplate.execute(status -> {  
        Picture picture = (Picture) stateContext.getExtendedState().getVariables().get(ContextKey.PICTURE_OBJ_KEY);  
  
        // 检查申诉额度  
        if (!userAppealQuotaService.canAppeal(picture.getUserId())) {  
            return false;  
        }  
  
        // 扣减申诉额度（事务保障下）  
        userAppealQuotaService.decreaseAppeal(picture.getUserId());  
  
        return true;  
    }));  
}
```
4. 动作
```Java
public Action<ImageReviewState, ImageReviewEvent> aiRejectAction() {  
    return context -> transactionTemplate.executeWithoutResult(status -> {  
        Picture picture = (Picture)context.getExtendedState().getVariables().get(ContextKey.PICTURE_OBJ_KEY);  
        if (picture == null) return;  
        log.debug("enter ai reject");  
        pictureService.markPictureWithStatus(picture.getId(), PictureReviewStatusEnum.FINAL_REJECTED.getValue(), null,"AI reject");  
        pictureService.warnUser(picture.getUserId());  
        String content = "您于 " + picture.getCreateTime() + " 发布的图片：" + picture.getName() + "经审核存在敏感内容！您被警告 1 次，超过三次将永久封禁！";  
        pushMessage(picture.getUserId(), content);  
    });  
}
```
5. 状态流转
```Java
@Configuration  
@EnableStateMachineFactory  
public class ImageReviewStateMachineConfig extends EnumStateMachineConfigurerAdapter<ImageReviewState, ImageReviewEvent> {  
  
    @Resource  
    private CommonActions actions;  
  
    @Resource  
    private Guards guards;  
  
    @Override  
    public void configure(StateMachineStateConfigurer<ImageReviewState, ImageReviewEvent> states) throws Exception {  
        states.withStates()  
                .initial(ImageReviewState.PENDING_REVIEW)  
                .states(EnumSet.allOf(ImageReviewState.class));  
    }  
  
    @Override  
    public void configure(StateMachineTransitionConfigurer<ImageReviewState, ImageReviewEvent> transitions) throws Exception {  
        transitions  
                // AI 审核阶段  
                .withExternal().source(ImageReviewState.PENDING_REVIEW).target(ImageReviewState.AI_PASS);
    }  
  
    @Override  
    public void configure(StateMachineConfigurationConfigurer<ImageReviewState, ImageReviewEvent> config) throws Exception {  
        config.withConfiguration().autoStartup(true);  
    }  
}
```

## Action 中涉及事务处理方案
在 Action 中不能使用 `@Transactional` 声明式注解，该注解利用 Spring AOP 动态代理开启事务，而 Action 中执行的动作并不会通过代理对象进行调用，因此声明式事务失效，需要使用编程式事务显示控制。