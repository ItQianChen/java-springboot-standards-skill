# 单体架构 vs 微服务架构形态对照与演进指南

本规范旨在建立一套统一的 Spring Boot 工程标准，使研发团队既能在**单体应用（Monolith / Modular Monolith）**中保持清晰低耦合的代码组织，又能在**微服务体系（Microservices）**与 **Spring AI 智能体体系**中实现无缝的分布式治理与横向扩展。

---

## 1. 架构形态全景对比矩阵

| 维度 | 单体架构 / 模块化单体 (Monolith) | 微服务体系 (Microservices) |
| :--- | :--- | :--- |
| **适用场景** | 中小型项目、初创业务验证、单进程低运维成本部署 | 大型复杂业务、多团队协作、高并发、需要独立弹性伸缩与 AI 赋能 |
| **Maven 组织形态** | 单一工程 或 轻量多模块 (`app`, `core`, `model`) | `parent` 统一版本管理 + `framework-*` 通用组件 + `api` 契约 + `gateway` 网关 + 独立业务服务 |
| **模块间调用** | Spring Bean 直接注入 / 内部接口调用 | Spring Cloud OpenFeign + Nacos 注册中心 + 负载均衡 |
| **解耦机制** | Spring `ApplicationEvent` 本地事件总线 | RabbitMQ 消息队列 (广播/延迟/死信队列) |
| **事务一致性** | Spring 本地事务 `@Transactional(rollbackFor = Exception.class)` | 本地事务 + Seata 全局分布式事务 (`@GlobalTransactional`) + MQ 最终一致性 |
| **并发与锁** | JVM `ReentrantLock` / Redisson 分布式锁 | Redisson 分布式锁 (`@Lock`) + Redis Lua 原子脚本 |
| **缓存策略** | 本地缓存 (Caffeine) 或 Redis + Spring Cache | Redis 多级缓存 + Spring Cache 多 CacheManager + Canal 缓存自动失效 |
| **检索与异构存储** | 数据库 Like 查询 或 单体整合 Elasticsearch | Canal 监听 MySQL Binlog 增量同步 -> Elasticsearch GeoDistance/全文检索 |
| **AI 与智能体集成** | 模块内注入 `ChatClient`，内存存储 `ChatMemory` | 独立 `aigc-service`，Multi-Agent 路由，@Tool 跨微服务 Feign 联动，`RedisChatMemory` 会话共享 |
| **流式响应 (SSE)** | Controller 直接输出 `Flux<T>`，加 `@NoWrapper` | 独立 AI 服务输出 SSE，网关保持长连接，前端解析 `DATA`/`PARAM` 事件 |
| **定时任务** | Spring `@Scheduled` / Spring Task | XXL-Job 分布式任务调度平台 (支持分片广播、超时告警、失败重试) |
| **数据库隔离** | 单一业务数据库，通过表前缀或逻辑模块隔离 | 微服务独立数据库，核心海量数据采用 ShardingSphere-JDBC 分库分表 |
| **网关与安全** | Spring Security / 统一 Filter 拦截鉴权 | Spring Cloud Gateway 统一路由 + JWT 解析 + 请求头注入 + 线程上下文传递 |

---

## 2. Maven 多模块工程结构对照

### 2.1 单体架构（模块化单体推荐结构）
```text
my-monolith-project/
├── pom.xml                               # 根 POM (依赖版本与公共 starter)
├── my-project-common/                    # 通用工具、异常、Result、PageResult、@NoWrapper
├── my-project-model/                     # 全局 Domain Entity、ReqDTO、ResDTO、VO
├── my-project-service/                   # Mapper、Service、业务逻辑、Spring AI Agent/Tool
└── my-project-web/                       # 启动类、Controller (按六端划分)、配置
```

### 2.2 微服务架构（生产级标准结构）
```text
my-microservice-project/
├── pom.xml                               # 根 POM
├── my-parent/                            # 版本统一管控 POM (管理 Spring Boot, Spring Cloud, Spring AI 版本)
├── my-framework/                         # 基础中间件与通用 Starter 集合
│   ├── my-common/                        # 核心工具、常量、异常、Result/PageResult 模型
│   ├── my-mvc/                           # 结果自动包装、全局异常拦截、UserContext 上下文
│   ├── my-mysql/                         # MyBatis-Plus 自动配置、PageUtils 分页工具
│   ├── my-redis/                         # Redisson @Lock、Spring Cache、RedisTemplate、Lua
│   ├── my-rabbitmq/                      # RabbitMQ 模板配置与消息模型
│   ├── my-es/                            # Elasticsearch Java Client 模板
│   ├── my-knife4j-web/                   # 接口文档 Starter
│   ├── my-statemachine/                  # 状态机持久化与流转组件
│   ├── my-canal-sync/                    # Canal 增量同步框架
│   ├── my-seata/                         # Seata 分布式事务 Starter
│   ├── my-sentinel/                      # 流量防卫与降级
│   └── my-xxl-job/                       # XXL-Job 执行器自动配置
├── my-api/                               # 跨服务契约层 (存放各服务的 FeignClient 接口与共享 DTO)
│   ├── customer-api/
│   ├── course-api/
│   ├── orders-api/
│   ├── trade-api/
│   └── interceptor/FeignInterceptor.java # Feign 请求头透传拦截器
├── my-gateway/                           # Spring Cloud Gateway (路由转发、JWT 校验、白名单)
└── my-services/                          # 独立部署的业务微服务
    ├── my-service-aigc/                  # Spring AI 智能体与流式对话服务
    ├── my-service-customer/
    ├── my-service-course/
    ├── my-service-orders/
    ├── my-service-market/
    ├── my-service-trade/
    └── my-service-publics/
```

---

## 3. 核心机制选型与实战代码对照

### 3.1 跨模块/服务调用与事件解耦

#### 单体架构方案：Spring Event 本地事件驱动
```java
// 1. 定义领域事件
@Getter
@AllArgsConstructor
public class OrderPayedEvent {
    private final Long orderId;
    private final Long userId;
}

// 2. 发布事件 (在订单支付业务中)
@Service
public class OrderServiceImpl implements IOrderService {
    @Resource
    private ApplicationEventPublisher eventPublisher;

    @Transactional(rollbackFor = Exception.class)
    public void paySuccess(Long orderId, Long userId) {
        // 更新订单状态...
        // 发布本地事件
        eventPublisher.publishEvent(new OrderPayedEvent(orderId, userId));
    }
}

// 3. 监听事件 (在积分/优惠券或通知模块中异步消费)
@Component
public class CouponEventListener {
    @Async
    @EventListener
    public void handleOrderPayed(OrderPayedEvent event) {
        // 处理积分奖励或卡券核销
    }
}
```

#### 微服务架构方案：OpenFeign + RabbitMQ 异步解耦
```java
// 1. Feign 契约定义 (放在 my-api 模块中)
@FeignClient(name = "service-market", path = "/market/inner/coupon")
public interface CouponApi {
    @PostMapping("/use")
    CouponUseResDTO use(@RequestBody CouponUseReqDTO reqDTO);
}

// 2. 跨服务事务调用 (Seata @GlobalTransactional)
@Service
public class OrderCreateServiceImpl implements IOrderCreateService {
    @Resource
    private CouponApi couponApi;
    @Resource
    private RabbitTemplate rabbitTemplate;

    @GlobalTransactional(rollbackFor = Exception.class)
    public void createOrderWithCoupon(OrderDTO orderDTO, Long couponId) {
        // 1. 远程核销优惠券 (RPC 同步调用)
        CouponUseResDTO couponRes = couponApi.use(new CouponUseReqDTO(orderDTO.getId(), couponId));
        // 2. 保存本地订单
        saveOrder(orderDTO, couponRes.getDiscountAmount());
        // 3. 发送 RabbitMQ 异步消息触发后续流程
        rabbitTemplate.convertAndSend(MqConstants.Exchanges.ORDER, MqConstants.RoutingKeys.ORDER_CREATED, new OrderMsg(orderDTO.getId()));
    }
}
```

---

## 4. 单体向微服务平滑演进路径

遵循本规范编写的单体应用，可零成本或极低成本拆分为微服务：
1. **边界不穿透**：单体阶段已按业务域划分包结构，并严格保持六端 Controller（`agency`, `consumer`, `inner`, `open`, `operation`, `worker`）隔离。
2. **接口先行**：单体内部的模块调用统一基于 Service 接口定义，拆分时仅需将接口迁移至 `api` 模块并增加 `@FeignClient` 注解。
3. **上下文透明**：统一依赖 `UserContext` 获取用户信息，单体阶段由 MVC 拦截器从 Session/JWT 填充，微服务阶段直接无缝由网关透传请求头填充。
4. **AI 工具解耦**：Spring AI 中的 `@Tool` 从单体阶段注入本地 Service，平滑迁移为微服务阶段注入 Feign Client，无需修改工具方法逻辑与提示词。
5. **数据解耦**：Domain Entity 与各端 DTO/VO 严格分离，数据库表不使用物理外键，拆分多库时无需重构数据传输模型。
---

## 5. 经典单体架构 (Modular Monolith) 生产级专项模式

基于经典三模块单体工程（如 `common + pojo + server` 体系）提炼的核心实战规范：

### 5.1 单体三模块分工标准
```text
<project-root>/
├── pom.xml                     # 根 POM (统一 Spring Boot、Lombok、MyBatis、Knife4j 等依赖版本)
├── <project>-common/           # 公共基础设施层
│   ├── constant/               # 常量类 (AutoFillConstant, MessageConstant, StatusConstant)
│   ├── context/                # ThreadLocal 上下文 (BaseContext / UserContext)
│   ├── enumeration/            # 公共枚举 (OperationType, UserType)
│   ├── exception/              # 自定义异常基类与派生业务异常 (BaseException, OrderBusinessException)
│   ├── json/                   # Jackson 序列化定制 (JacksonObjectMapper 日期时间格式化)
│   ├── properties/             # 外部组件属性配置 (@ConfigurationProperties)
│   ├── result/                 # 统一响应对象 (Result<T>, PageResult<T>)
│   └── utils/                  # 工具类 (JwtUtil, AliOssUtil, HttpClientUtil, WeChatPayUtil)
├── <project>-pojo/             # 数据传输与领域模型层 (全模块共享，杜绝循环依赖)
│   ├── entity/                 # 数据库实体 (@Data)
│   ├── dto/                    # 前端请求入参 (*DTO, *PageQueryDTO)
│   └── vo/                     # 前端展示视图对象 (*VO, *ReportVO)
└── <project>-server/           # 业务聚合实现与启动模块
    ├── annotation/             # 业务注解 (@AutoFill)
    ├── aspect/                 # AOP 切面 (AutoFillAspect)
    ├── config/                 # 配置类 (WebMvcConfiguration, RedisConfiguration, WebSocketConfiguration)
    ├── controller/             # 控制器 (严格按端隔离: admin/, user/)
    ├── handler/                # 全局异常拦截器 (GlobalExceptionHandler)
    ├── interceptor/            # 各端鉴权拦截器 (JwtTokenAdminInterceptor, JwtTokenUserInterceptor)
    ├── mapper/                 # MyBatis Mapper 接口与 XML 文件
    ├── service/impl/           # 业务逻辑接口与实现
    ├── task/                   # Spring Task 本地定时任务 (OrdersTask)
    └── websocket/              # WebSocket 服务端实现 (WebSocketServer)
```

### 5.2 单体无网关环境下的多端独立鉴权拦截器
单体项目通常在一个应用中同时承载管理端（Admin/Operation）与用户端（User/Consumer），必须通过双/多拦截器实现端级认证隔离：

```java
@Configuration
public class WebMvcConfiguration extends WebMvcConfigurationSupport {

    @Resource
    private JwtTokenAdminInterceptor jwtTokenAdminInterceptor;
    @Resource
    private JwtTokenUserInterceptor jwtTokenUserInterceptor;

    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        // 1. 管理端拦截器：挂载 /admin/**，放行登录等免密端点
        registry.addInterceptor(jwtTokenAdminInterceptor)
                .addPathPatterns("/admin/**")
                .excludePathPatterns("/admin/employee/login");

        // 2. 用户端拦截器：挂载 /user/**，放行用户登录、公开店铺状态等
        registry.addInterceptor(jwtTokenUserInterceptor)
                .addPathPatterns("/user/**")
                .excludePathPatterns("/user/user/login", "/user/shop/status");
    }
}
```

### 5.3 AOP + 自定义注解实现公共字段自动填充
在纯 MyBatis 或需要通用切面赋值的场景下，通过 `@AutoFill` 注解配合 AOP 前置通知完成创建/更新时间和操作人的自动注入：

```java
// 1. 定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoFill {
    OperationType value(); // INSERT 或 UPDATE
}

// 2. 切面统一赋值
@Aspect
@Component
public class AutoFillAspect {
    @Pointcut("execution(* com.company..mapper.*.*(..)) && @annotation(com.company.annotation.AutoFill)")
    public void autoFillPointCut() {}

    @Before("autoFillPointCut()")
    public void autoFill(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class);
        OperationType operationType = autoFill.value();

        Object[] args = joinPoint.getArgs();
        if (args == null || args.length == 0) return;
        Object entity = args[0];

        LocalDateTime now = LocalDateTime.now();
        Long currentId = BaseContext.getCurrentId();

        try {
            if (operationType == OperationType.INSERT) {
                Method setCreateTime = entity.getClass().getDeclaredMethod("setCreateTime", LocalDateTime.class);
                Method setCreateUser = entity.getClass().getDeclaredMethod("setCreateUser", Long.class);
                Method setUpdateTime = entity.getClass().getDeclaredMethod("setUpdateTime", LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod("setUpdateUser", Long.class);
                setCreateTime.invoke(entity, now);
                setCreateUser.invoke(entity, currentId);
                setUpdateTime.invoke(entity, now);
                setUpdateUser.invoke(entity, currentId);
            } else if (operationType == OperationType.UPDATE) {
                Method setUpdateTime = entity.getClass().getDeclaredMethod("setUpdateTime", LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod("setUpdateUser", Long.class);
                setUpdateTime.invoke(entity, now);
                setUpdateUser.invoke(entity, currentId);
            }
        } catch (Exception e) {
            log.error("公共字段自动填充失败: {}", e.getMessage(), e);
        }
    }
}
```

### 5.4 Knife4j / Swagger 多端独立文档分组 (Multi-Docket)
在单体多端架构下，通过配置多个 `Docket` Bean 按包路径进行文档物理隔离：

```java
@Bean
public Docket adminApiDocket() {
    return new Docket(DocumentationType.SWAGGER_2)
            .groupName("管理端接口")
            .apiInfo(new ApiInfoBuilder().title("管理端接口文档").version("1.0").build())
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.company.controller.admin"))
            .paths(PathSelectors.any())
            .build();
}

@Bean
public Docket userApiDocket() {
    return new Docket(DocumentationType.SWAGGER_2)
            .groupName("用户端接口")
            .apiInfo(new ApiInfoBuilder().title("用户端接口文档").version("1.0").build())
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.company.controller.user"))
            .paths(PathSelectors.any())
            .build();
}
```

### 5.5 全局日期时间序列化扩展 (Jackson Mapping)
通过扩展 `MappingJackson2HttpMessageConverter`，统一前后端日期时间格式，彻底消除反序列化乱码与格式异常：

```java
public class JacksonObjectMapper extends ObjectMapper {
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    public JacksonObjectMapper() {
        super();
        SimpleModule simpleModule = new SimpleModule()
                .addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)))
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        this.registerModule(simpleModule);
    }
}
```

### 5.6 单体轻量全双工通讯 (Spring WebSocket)
在单体应用中，实现来单语音播报、用户催单等即时推送，使用 Spring 原生 `@ServerEndpoint` 即可完成，无需额外引入 MQ：

```java
@Component
@ServerEndpoint("/ws/{sid}")
public class WebSocketServer {
    private static final Map<String, Session> SESSION_MAP = new ConcurrentHashMap<>();

    @OnOpen
    public void onOpen(Session session, @PathParam("sid") String sid) {
        SESSION_MAP.put(sid, session);
    }

    @OnClose
    public void onClose(@PathParam("sid") String sid) {
        SESSION_MAP.remove(sid);
    }

    // 群发消息 (如订单提醒: type=1 来单提醒, type=2 客户催单)
    public void sendToAllClient(String message) {
        for (Session session : SESSION_MAP.values()) {
            try {
                session.getBasicRemote().sendText(message);
            } catch (Exception e) {
                log.error("WebSocket 消息推送失败: {}", e.getMessage(), e);
            }
        }
    }
}
```
