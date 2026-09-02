# 核心设计模式、高级中间件与 Spring AI 智能体生产范式

本规范汇集了企业级大型微服务项目（云岚到家、天机学堂 AI 体系）中经过高并发与智能交互检验的**核心设计模式**、**分布式中间件**与 **Spring AI 智能体开发范式**。

---

## 1. Spring AI 与智能体微服务架构标准 (Spring AI & Multi-Agent)

在融合大模型（LLM）的 Spring Boot 微服务架构中，AI 能力统一沉淀在独立的 `aigc-service`（或单体中的 `agent` 模块），并通过 **Multi-Agent 路由、Feign Tool 工具链调用、SSE 结构化事件流、分布式会话记忆与 Token 优化 Advisor** 实现生产级闭环。

```text
                                  [ 用户提问 /chat (SSE Stream) ]
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │      RouteAgent       │ (意图识别与路由智能体)
                                    └───────────┬───────────┘
                                                │
                ┌───────────────────┬───────────┴───────────┬───────────────────┐
                ▼                   ▼                       ▼                   ▼
        ┌───────────────┐   ┌───────────────┐       ┌───────────────┐   ┌───────────────┐
        │ ConsultAgent  │   │ KnowledgeAgent│       │ RecommendAgent│   │   BuyAgent    │
        │  (课程咨询)   │   │  (RAG知识库)  │       │  (课程推荐)   │   │  (下单助手)   │
        └───────┬───────┘   └───────┬───────┘       └───────┬───────┘   └───────┬───────┘
                │                   │                       │                   │
                ▼                   ▼                       ▼                   ▼
    ┌───────────────────────────────────────────────────────────────────────────────────┐
    │                         微服务 @Tool 工具层 (注入各业务 FeignClient)                 │
    │  CourseTools.queryCourseById(...)  ──调用──>  CourseClient.baseInfo(...)           │
    │  OrderTools.prePlaceOrder(...)     ──调用──>  OrderClient.create(...)              │
    └───────────────────────────────────────────────────────────────────────────────────┘
                │
                ▼ (数据透传: ToolContext + ToolResultHolder)
    ┌───────────────────────────────────────────────────────────────────────────────────┐
    │                      SSE 结构化事件流输出 (MediaType.TEXT_EVENT_STREAM)             │
    │   • DATA  : 大模型流式输出文本 Token                                                │
    │   • PARAM : 结构化业务数据卡片 (如课程卡片、预下单详情，直接给前端渲染富组件)       │
    │   • STOP  : 结束信号                                                              │
    └───────────────────────────────────────────────────────────────────────────────────┘
```

### 1.1 Multi-Agent 抽象基类与流式响应生命周期 (`AbstractAgent`)
所有 Agent 继承统一抽象基类，规范生命周期监听、中断控制与会话记忆：

```java
@Slf4j
public abstract class AbstractAgent implements Agent {
    @Resource
    private ChatClient dashScopeChatClient;
    @Resource
    private ChatMemory chatMemory;
    @Resource
    private ChatSessionService chatSessionService;

    // 内存生成状态 (生产环境集群推荐使用 Redis 存取)
    public static final Map<String, Boolean> GENERATE_STATUS = new ConcurrentHashMap<>();
    public static final ChatEventVO STOP_EVENT = ChatEventVO.builder().eventType(ChatEventTypeEnum.STOP.getValue()).build();

    public Flux<ChatEventVO> processStream(String question, String sessionId) {
        Long userId = UserContext.currentUserId();
        String requestId = IdUtil.fastSimpleUUID();
        StringBuilder outputBuilder = new StringBuilder();

        // 1. 刷新会话最近活动时间
        this.chatSessionService.addOrUpdate(sessionId, question, userId);

        // 2. 构造并执行流式请求
        return this.getChatClientRequest(sessionId, requestId, question)
                .stream()
                .chatResponse()
                .doFirst(() -> GENERATE_STATUS.put(sessionId, true))        // 标记输出中
                .doOnComplete(() -> GENERATE_STATUS.remove(sessionId))      // 完成清理标记
                .doOnError(e -> GENERATE_STATUS.remove(sessionId))         // 异常清理标记
                .doOnCancel(() -> {
                    // 用户主动取消/中断时，将已生成的部分内容落盘保存到历史会话
                    saveStopHistoryRecord(sessionId, outputBuilder.toString());
                    GENERATE_STATUS.remove(sessionId);
                })
                .takeWhile(s -> Optional.ofNullable(GENERATE_STATUS.get(sessionId)).orElse(false)) // 支持外部主动打断
                .map(chatResponse -> {
                    // 处理流式文本 Chunk
                    String text = chatResponse.getResult().getOutput().getText();
                    outputBuilder.append(text);
                    return ChatEventVO.builder()
                            .eventType(ChatEventTypeEnum.DATA.getValue())
                            .eventData(text)
                            .build();
                })
                .concatWith(Flux.defer(() -> {
                    // 3. 提取 Tool 执行过程中产生的结构化业务参数并回传前端
                    Map<String, Object> toolParams = ToolResultHolder.get(requestId);
                    if (CollUtil.isNotEmpty(toolParams)) {
                        ToolResultHolder.remove(requestId);
                        ChatEventVO paramEvent = ChatEventVO.builder()
                                .eventType(ChatEventTypeEnum.PARAM.getValue())
                                .eventData(toolParams)
                                .build();
                        return Flux.just(paramEvent, STOP_EVENT);
                    }
                    return Flux.just(STOP_EVENT);
                }));
    }

    private ChatClient.ChatClientRequestSpec getChatClientRequest(String sessionId, String requestId, String question) {
        return this.dashScopeChatClient.prompt()
                .system(sp -> sp.text(this.systemMessage()).params(this.systemMessageParams()))
                .advisors(adv -> adv.advisors(this.advisors()).params(this.advisorParams(sessionId, requestId)))
                .tools(this.tools())
                .toolContext(Map.of("sessionId", sessionId, "requestId", requestId))
                .user(question);
    }
}
```

### 1.2 微服务联动 Tool 标准规范 (`@Tool` + `FeignClient`)
AI 工具层直接注入微服务 Feign Client，利用 `ToolContext` 与 `ToolResultHolder` 穿透机制，将后端业务数据既提供给大模型推理，又无损透传给前端渲染富 UI 组件：

```java
@Component
@RequiredArgsConstructor
public class CourseTools {
    private final CourseClient courseClient; // 内部微服务 Feign 客户端

    @Tool(description = "根据课程ID查询课程的详细基本信息、讲师与价格")
    public CourseInfo queryCourseById(@ToolParam(description = "课程ID") Long courseId, ToolContext toolContext) {
        if (courseId == null) {
            return null;
        }
        // 1. 远程调用课程微服务获取实体
        CourseDTO courseDTO = courseClient.baseInfo(courseId, true);
        CourseInfo courseInfo = CourseInfo.of(courseDTO);

        // 2. 将结构化对象存入 ToolResultHolder，供 SSE 流结束时以 PARAM 事件发送给前端
        String requestId = (String) toolContext.getContext().get("requestId");
        ToolResultHolder.put(requestId, "courseInfo_" + courseId, courseInfo);

        // 3. 返回给大模型用于推理决策
        return courseInfo;
    }
}
```

### 1.3 SSE 控制器与 `@NoWrapper` 穿透注解
流式响应方法必须使用 `@NoWrapper`（防止 MVC Filter 包装为 `Result<T>`）并标明 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`：

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/chat")
public class ChatController {
    private final ChatService chatService;

    @NoWrapper // 关键：禁用统一 JSON 包装，确保 SSE 原始流式传输
    @PostMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatEventVO> chat(@RequestBody ChatDTO dto) {
        return chatService.chat(dto);
    }

    @PostMapping("/stop")
    public void stop(@RequestParam("sessionId") String sessionId) {
        chatService.stop(sessionId);
    }
}
```

### 1.4 分布式会话记忆与 Advisor 拦截优化 (`RedisChatMemory` & `RecordOptimizationAdvisor`)
* **`RedisChatMemory`**：实现 Spring AI `ChatMemory` 接口，使用 Redis 列表存储 `List<Message>`，保证微服务集群多 Pod 间会话记忆共享。
* **`RecordOptimizationAdvisor`**：在 `aroundCall` 与 `aroundStream` 中过滤中间路由消息、对历史对话进行滑动窗口修剪与 Token 压缩，降低模型上下文消耗。

---

## 2. 分布式锁标准范式 (`Redisson` / `@Lock`)

### 2.1 注解式分布式锁 (`@Lock`)
```java
@Override
@Lock(formatter = RedisConstants.RedisFormatter.SEIZE, time = 300)
public void seize(Long id, Long serveProviderId, Integer serveProviderType, Boolean isMachine) {
    // 业务临界区操作...
}
```

### 2.2 编程式分布式锁范式
```java
@Resource
private RedissonClient redissonClient;

public void processWithLock(String lockKey) {
    RLock lock = redissonClient.getLock(lockKey);
    try {
        boolean acquired = lock.tryLock(3, 10, TimeUnit.SECONDS);
        if (!acquired) {
            throw new CommonException(ErrorInfo.Code.FORBIDDEN_OPERATION, "系统繁忙，请稍后重试");
        }
        // 执行受保护的业务...
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new CommonException(ErrorInfo.Code.PROCESS_FAILD, "加锁被中断");
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

---

## 3. 状态机驱动业务流转模式 (Spring StateMachine)

复杂状态流转（如订单、支付、退款、审批）严禁直接 `update status = ?`，必须通过状态机统一驱动：

```java
@Resource
private OrderStateMachine orderStateMachine;

// 1. 启动状态机
OrderSnapshotDTO snapshot = BeanUtil.toBean(orders, OrderSnapshotDTO.class);
snapshot.setPayStatus(OrderPayStatusEnum.NO_PAY.getStatus());
orderStateMachine.start(orders.getUserId(), orders.getId().toString(), snapshot);

// 2. 驱动状态变更
OrderSnapshotDTO paySnapshot = OrderSnapshotDTO.builder()
        .payTime(LocalDateTime.now())
        .tradingOrderNo(tradeMsg.getTradingOrderNo())
        .payStatus(OrderPayStatusEnum.PAY_SUCCESS.getStatus())
        .build();
orderStateMachine.changeStatus(orders.getUserId(), String.valueOf(orders.getId()), OrderStatusChangeEventEnum.PAYED, paySnapshot);
```

---

## 4. 高并发 Redis + Lua 原子操作范式

在秒杀、抢单、抢券等超高并发场景下，直接操作数据库将导致行锁竞争与性能雪崩。必须通过 **Redis + Lua 脚本**在内存中完成资格校验、库存扣减与异步队列写入：

```java
@Resource(name = "seizeOrdersScript")
private DefaultRedisScript<String> seizeOrdersScript;
@Resource
private RedisTemplate redisTemplate;

public void executeLuaSeize(Long orderId, Long workerId, Integer workerType, int cityIndex) {
    String syncQueueKey = RedisSyncQueueUtils.getQueueRedisKey(RedisConstants.RedisKey.ORDERS_SEIZE_SYNC_QUEUE_NAME, cityIndex);
    String stockKey = String.format(RedisConstants.RedisKey.ORDERS_RESOURCE_STOCK, cityIndex);
    String workerStateKey = String.format(RedisConstants.RedisKey.SERVE_PROVIDER_STATE, cityIndex);

    Object result = redisTemplate.execute(
            seizeOrdersScript,
            new GenericJackson2JsonRedisSerializer(),
            new GenericJackson2JsonRedisSerializer(),
            Arrays.asList(syncQueueKey, stockKey, workerStateKey),
            orderId, workerId, workerType
    );

    if (result == null || NumberUtils.parseLong(result.toString()) < 0) {
        throw new CommonException(ErrorInfo.Code.SEIZE_ORDERS_FAILD, "抢单失败，资源已被抢占");
    }
}
```

---

## 5. 业务层核心设计模式应用

### 5.1 策略模式 + 工厂模式 (多渠道支付)
```java
// 策略接口
public interface BasicPayHandler {
    NativePayResDTO createDownLineTrading(NativePayReqDTO reqDTO);
}

// 策略实现
@Component
@PayChannel(type = PayChannelEnum.WECHAT_PAY)
public class WechatNativePayHandler implements BasicPayHandler {
    @Override
    public NativePayResDTO createDownLineTrading(NativePayReqDTO reqDTO) { ... }
}

// 策略工厂
@Component
public class HandlerFactory {
    private final Map<PayChannelEnum, BasicPayHandler> handlerMap = new ConcurrentHashMap<>();

    @Autowired
    public HandlerFactory(List<BasicPayHandler> handlers) {
        for (BasicPayHandler handler : handlers) {
            PayChannel annotation = handler.getClass().getAnnotation(PayChannel.class);
            if (annotation != null) {
                handlerMap.put(annotation.type(), handler);
            }
        }
    }

    public BasicPayHandler get(PayChannelEnum channel) {
        return handlerMap.get(channel);
    }
}
```

### 5.2 规则管道模式 (智能派单引擎)
```java
public interface IRule {
    void filter(RuleContext context);
}

@Component
public class DispatchRuleRunner {
    @Resource
    private List<IRule> rules;

    public List<Long> evaluateCandidates(RuleContext context) {
        for (IRule rule : rules) {
            rule.filter(context);
            if (context.getCandidateIds().isEmpty()) {
                break; // 提前短路
            }
        }
        return context.getCandidateIds();
    }
}
```

---

## 6. 分布式事务 (Seata `@GlobalTransactional`)

```java
@Service
public class OrdersCreateServiceImpl implements IOrdersCreateService {
    @Resource
    private CouponApi couponApi;
    @Resource
    private IOrdersCreateService owner;

    @GlobalTransactional(rollbackFor = Exception.class)
    @Override
    public void addWithCoupon(Orders orders, Long couponId) {
        // 1. 远程核销卡券 (RPC)
        CouponUseResDTO couponRes = couponApi.use(new CouponUseReqDTO(orders.getId(), couponId, orders.getTotalAmount()));
        // 2. 本地装配与落盘
        orders.setDiscountAmount(couponRes.getDiscountAmount());
        orders.setRealPayAmount(orders.getTotalAmount().subtract(couponRes.getDiscountAmount()));
        save(orders);
    }
}
```

---

## 7. 异构数据增量同步与搜索 (Canal + ElasticSearch)

* **Canal 监听 Binlog**：实时捕获数据库变更事件。
* **数据加工与 ES 写入**：由 `CanalDataSyncHandler` 组装复合维度数据并刷新至 ElasticSearch。
* **Elasticsearch 检索**：采用 `ElasticSearchTemplate` 实现 `GeoDistance` 距离范围过滤、条件组合与 `searchAfter` 滚动分页。

---

## 8. 分布式任务调度 (XXL-Job)

```java
@Component
@Slf4j
public class XxlJobHandler {
    @XxlJob("cancelOvertimeOrdersJob")
    public void cancelOvertimeOrdersJob() {
        int shardIndex = XxlJobHelper.getShardIndex();
        int shardTotal = XxlJobHelper.getShardTotal();
        log.info("执行超时订单清理, 分片: {}/{}", shardIndex, shardTotal);
        // 执行分片业务逻辑...
    }
}
```

---

## 9. OpenFeign 拦截器与上下文透传

```java
@Component
public class FeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        CurrentUserInfo currentUser = UserContext.currentUser();
        if (currentUser != null) {
            template.header(HeaderConstants.USER_ID, String.valueOf(currentUser.getId()));
            template.header(HeaderConstants.USER_TYPE, String.valueOf(currentUser.getUserType()));
        }
        String requestId = RequestContext.getRequestId();
        if (StringUtils.isNotEmpty(requestId)) {
            template.header(HeaderConstants.REQUEST_ID, requestId);
        }
    }
}
```