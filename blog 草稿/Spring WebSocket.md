WebSocket 协议本质上运行在 TCP/IP 之上，握手过程借助了 HTTP 协议来建立全双工的 WebSocket 帧传输连接。

WebSocketSession 的生命周期由 Spring 框架管理（创建、关闭时回调），但是用户和会话的绑定需要手动处理，比如利用 Map 集合存储用户 ID 及其对应的 WebSocketSession。同时 WebSocket 连接健康状态需要手动管理。

# Spring WebSocket 核心组成
Spring 整合 WebSocket 核心包括 **拦截器（HandshakeInterceptor）**、**处理器（WebSocketHandler）** 和 **配置（WebSocketConfigurer）**。

**1. 拦截器（HandshakeInterceptor）**
作用于HTTP 握手阶段，在升级为 WebSocket 前后执行逻辑。
**2. 处理器（WebSocketHandler / TextWebSocketHandler）**
- 作用于WebSocket 连接建立之后，处理消息收发及生命周期回调。
- 核心方法：
    - `afterConnectionEstablished(WebSocketSession session)`：连接建立成功
    - `handleMessage(WebSocketSession session, WebSocketMessage<?> message)`：处理消息
    - `afterConnectionClosed(WebSocketSession session, CloseStatus status)`：连接关闭
**3. 配置（WebSocketConfigurer）**
注册 `WebSocketHandler`、配置 URL 映射、添加拦截器。

# WebSocketSession 管理
WebSocketSession 不包含业务信息，需要手段管理用户及其对应的 WebSocketSession，常见的做法是利用 Map 存储用户 Id 和 Session 键值对。

```Java
@Component  
public class UserConnectionManager {  
    private final ConcurrentHashMap<Long, UserConnectionState> connections = new ConcurrentHashMap<>();  
  
    public synchronized boolean register(Long userId, ConnectionType type, Object connectionRef) {  
        UserConnectionState existing = connections.get(userId);  
        // 优先使用 WebSocket，如果已有 WebSocket 则忽略 SSE        if (existing != null && existing.getType() == ConnectionType.WEBSOCKET && type == ConnectionType.SSE) {  
            return false; // 拒绝降级注册  
        }  
        connections.put(userId, new UserConnectionState(userId, type, connectionRef, Instant.now()));  
        return true;  
    }  
  
    // 断开连接  
    public void unregister(Long userId) {  
        connections.remove(userId);  
    }  
  
    // 判断是否在线  
    public boolean isOnline(Long userId) {  
        return connections.containsKey(userId);  
    }  
  
    // 获取连接类型  
    public ConnectionType getConnectionType(Long userId) {  
        return Optional.ofNullable(connections.get(userId)).map(UserConnectionState::getType).orElse(null);  
    }  
  
    // 获取连接对象（用于推送）  
    public Object getConnectionRef(Long userId) {  
        return Optional.ofNullable(connections.get(userId)).map(UserConnectionState::getConnectionRef).orElse(null);  
    }  
  
    public void updateHeartbeat(Long userId) {  
        UserConnectionState state = connections.get(userId);  
        if (state != null) {  
            state.setLastActive(Instant.now());  
        }  
    }  
  
    public ConcurrentHashMap<Long, UserConnectionState> getAllConnections() {  
        return connections;  
    }  
}
```
# WebSocket 连接健康管理
用户掉线后，可能并不会触发 afterConnectionClosed() 回调，导致服务端长期保留无用的 WebSocket 连接，造成资源浪费，因此需要引入连接健康管理。

管理 WebSocket 连接健康有两种方案：
1. 使用 WebSocket 协议原生 ping/pong
	WebSocket 标准定义了 ping/pong 控制帧，轻量且不需要额外编码协议。但是客户端只能是浏览器，多数浏览器原生 WebSocket 自动处理 ping/pong，其他客户端需要手写。且只能是服务端 ping，客户端 pong。
2. 自定义实现 ping/pong
	使用 JSON 或字符串消息，服务端在 `handleTextMessage()` 收到后更新最后活跃时间，如果超时未收到则关闭连接，定时任务检测超时连接并调用 `session.close()` 清理。

第一种方案仅支持浏览器客户端，第二种方案可以适用其他客户端。

```Java
/**
* 自定义用户连接状态
*/
@Data  
@AllArgsConstructor  
@NoArgsConstructor  
public class UserConnectionState {  
    private Long userId;  
    private ConnectionType type;  
    private Object connectionRef;  
    private Instant lastActive;  
}
```

```Java
// 服务端处理客户端发来的 ping 消息
@Override  
public void handleTextMessage(@Nonnull WebSocketSession session, TextMessage message) throws Exception {  
    String payload = message.getPayload();  
    MessageInfo me = JSONUtil.toBean(payload, MessageInfo.class);  
    if ("ping".equalsIgnoreCase(me.getType())) {  
        Long userId = getUserIdFromSession(session);  
        connectionManager.updateHeartbeat(userId);  
        // 回复 pong 给前端，确认连接活跃  
        session.sendMessage(new TextMessage("pong"));  
    }  
}
```

```Java
/**
* 更新心跳时间
*/
public void updateHeartbeat(Long userId) {  
	UserConnectionState state = connections.get(userId);  
	if (state != null) {  
		state.setLastActive(Instant.now());  
	}  
}  
```

```Java
/**
* 定时任务扫描 Map 集合，将长时间未响应的 session 移除并关闭连接。
*/
@Component  
public class HeartbeatMonitor {  
  
    @Resource  
    private UserConnectionManager connectionManager;  
  
    @Value("${app.heartbeat}")  
    private long maxInactiveSeconds;  
  
    private Duration maxInactiveDuration;  
  
    @PostConstruct  
    public void init() {  
        this.maxInactiveDuration = Duration.ofSeconds(maxInactiveSeconds);  
    }  
  
    @Scheduled(fixedDelay = 30000)  
    public void checkHeartbeat() {  
        Instant now = Instant.now();  
        for (Map.Entry<Long, UserConnectionState> entry : connectionManager.getAllConnections().entrySet()) {  
            Long userId = entry.getKey();  
            UserConnectionState state = entry.getValue();  
            if (Duration.between(state.getLastActive(), now).compareTo(maxInactiveDuration) > 0) {  
                Object conn = state.getConnectionRef();  
                if (conn instanceof WebSocketSession) {  
                    try {  
                        ((WebSocketSession) conn).close(CloseStatus.GOING_AWAY);  
                    } catch (IOException e) {  
                        // 忽略  
                    }  
                }  
                connectionManager.unregister(userId);  
            }  
        }  
    }  
}
```

思考：
当存在大量 WebSocket 连接时，扫描一次的耗时有可能超过定时任务的间隔时间，并且大量 WebSocketSession 直接存储在内存是否会导致溢出？

# WebSocket 消息推送实践

用户上传图片后，图片进入审核通道，需要将审核结果通知给用户，也就是需要服务端推送消息给用户。

参考 bilibili 收件箱，实现一个小规模可用的消息推送方案。消息推送时，首先就要考虑用户是否在线。如果用户在线，则可以 WebSocket 结合其他方案将消息实时的反馈给用户，否则，当用户登录上线建立 WebSocket 连接后，立即发送提示消息引导用户前往个人页面收件箱，此时客户端主动的拉取消息列表并展示给用户。如此，便实现了消息在线/离线全场景推送。

目前系统仅提供了审核消息的推送，但考虑到后续可能会增加点赞提示等等涉及到消息推送的场景，所以涉及了一个可以拓展的消息推送模块。只需实现统一的消息接口，并在消费者侧新增处理逻辑，即可拓展新的推送逻辑。

## 在线消息推送
考虑到消息数量一般来说比较庞大，且消息推送耗时一般较长，所以使用消息队列解耦了消息推送服务和其他服务。

在线消息推送结合了 WebSocket 和 SSE(Server-Sent Event) ，WebSocket 优先，当建立 WebSocket 连接失败后，使用 SSE 推送消息。无论是使用 WebSocket 还是 SSE，都需要一个统一的用户连接管理器管理连接的注册、移除等内容。

WebSocket 需要手动管理连接健康状态而 SSE 有浏览器端的自动重连机制，可以保证客户端连接“尽量可用”，**SSE 单向推送，没有 ping/pong**，需要自己推送心跳 event，如果客户端断开，服务端在下一次写入时就会抛出异常，立刻感知。

由于涉及在线和离线场景，所以消息需要持久化到数据库。

消息发送流程可以概括为：
1. 业务端持久化消息（未读）并发送消息到消息队列
2. 消息推送端接收消息开始处理
3. 根据消息类型进行处理
4. 从连接管理器获取用户连接状态，如果在线，则直接推送，否则不做处理。
5. 用户阅读消息后点击确认，调用服务端接口，将消息状态改为已读。

## 离线消息拉取
成功建立 WebSocket 连接后，发送消息引导用户点击收件箱，客户端主动拉取未读信息到收件箱。
