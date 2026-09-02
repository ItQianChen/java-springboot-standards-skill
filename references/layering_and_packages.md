# 标准包结构、六端 Controller 分离与数据流转规范

本规范定义了 Spring Boot 单体与微服务工程中各层的目录职责、六端 Controller 物理隔离体系、Spring AI 智能体子模块结构以及领域模型（Domain/DTO/VO）的全生命周期流转规则。

---

## 1. 业务与 AI 工程通用包结构

所有业务服务或 AI 模块均须遵循如下标准包结构，保持职责单一与层级清晰。

**【按需裁剪铁律】**：`agent/`、`advisor/`、`memory/`、`tools/` 仅在工程明确引入了 `spring-ai` 或用户有 AI 需求时按需创建；**普通的单体项目或传统微服务项目严禁创建任何 AI 相关目录和类！**

```text
com.<company>.<project/service>
├── advisor/                    # [仅AI项目按需] Spring AI 请求与流式拦截器 (Token优化/日志/上下文)
│   └── RecordOptimizationAdvisor.java
├── agent/                      # [仅AI项目按需] 智能体体系与意图路由
│   ├── AbstractAgent.java      # 智能体通用抽象基类 (管理流式生命周期、会话、ToolContext)
│   ├── RouteAgent.java         # 意图识别与路由智能体
│   ├── ConsultAgent.java       # 业务咨询智能体
│   └── KnowledgeAgent.java     # RAG 知识库检索智能体
├── client/                     # 外部 HTTP/三方 API 客户端调用与封装 (或 Feign Client)
│   └── AmapHttpClient.java
├── config/                     # Spring 配置类 (@Configuration, @Bean 注入)
│   ├── SpringAIConfig.java     # (仅AI项目)
│   └── SecurityConfig.java
├── constants/                  # 业务常量与缓存 Key 定义 (严禁硬编码)
│   ├── RedisConstants.java
│   └── ErrorInfo.java
├── controller/                 # 控制器层 (严格按六端物理分包)
│   ├── agency/                 # 机构端 / B端商户端接口 (/agency/**)
│   ├── consumer/               # C端用户 / 小程序端 / APP端接口 (/consumer/**)
│   ├── inner/                  # 内部/跨服务 Feign 接口 (/inner/**)
│   ├── open/                   # 开放免认证接口 (/open/**，如登录、短信验证码)
│   ├── operation/              # 运营端 / 平台管理后台接口 (/operation/**)
│   └── worker/                 # 服务人员端 / 师傅履约端接口 (/worker/**)
├── enums/                      # 业务状态、类型与状态机事件枚举
│   ├── OrderStatusEnum.java
│   └── ChatEventTypeEnum.java
├── handler/                    # 任务处理器与数据同步器
│   ├── XxlJobHandler.java
│   └── CanalDataSyncHandler.java
├── listener/                   # 异步消息监听器 (RabbitMQ / Spring Event)
│   └── OrderPayStatusListener.java
├── mapper/                     # MyBatis-Plus BaseMapper 接口与对应 XML 文件
│   └── OrdersMapper.java
├── memory/                     # [仅AI项目按需] 会话记忆实现 (如 RedisChatMemory)
│   └── RedisChatMemory.java
├── model/                      # 领域模型 (分层对象严禁混用)
│   ├── domain/                 # 数据库持久化实体类 (@TableName, @TableId)
│   ├── dto/                    # 数据传输对象
│   │   ├── request/            # 请求参数 DTO (*ReqDTO, *PageQueryReqDTO)
│   │   └── response/           # 响应数据 DTO (*ResDTO)
│   └── vo/                     # 页面/视图展示对象 (*VO, ChatEventVO)
├── properties/                 # 配置属性映射类 (@ConfigurationProperties)
│   └── ApplicationProperties.java
├── service/                    # 业务逻辑接口定义 (I*Service)
│   ├── IOrdersService.java
│   └── impl/                   # 业务逻辑接口实现 (*ServiceImpl)
│       └── OrdersServiceImpl.java
├── strategy/                   # 策略模式实现与规则管道
│   └── ScoreStrategy.java
└── tools/                      # [仅AI项目按需] Spring AI @Tool 工具链实现与微服务联动
    ├── CourseTools.java        # 工具类 (@Tool 方法注入 Feign Client)
    └── result/                 # 工具调用产出的结构化数据模型 (如 CourseInfo, PrePlaceOrder)
```

---

## 2. 六端 Controller 物理隔离规范

企业级业务系统通常服务于多种不同的用户角色。为杜绝权限越权、接口混乱及安全漏洞，Controller 层必须严格按业务端进行物理分包与路由前缀隔离：

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                             六端 CONTROLLER 隔离架构                         │
└────────────────────────────────────────────────────────────────────────────┘
         │
         ├── 1. agency/      (/agency/**)    --> 机构端/加盟商/B端商户操作与管理
         ├── 2. consumer/    (/consumer/**)  --> 客户端/C端小程序/移动端下单与查询
         ├── 3. inner/       (/inner/**)     --> 内部跨模块/Feign专用接口 (网关禁止外网暴露)
         ├── 4. open/        (/open/**)      --> 开放端/免登录接口 (验证码、登录、公开信息)
         ├── 5. operation/   (/operation/**) --> 运营端/平台管理后台/审核与配置管理
         └── 6. worker/      (/worker/**)    --> 服务人员端/师傅端/抢单与履约执行
```

### 2.1 隔离规则与安全约束
1. **路由前缀一致性**：Controller 类上的 `@RequestMapping` 必须与所在包名保持严格一致（如 `controller/consumer/AddressBookController.java` 路由必须为 `@RequestMapping("/consumer/address-book")`）。
2. **Spring Bean 命名防重**：不同端存在同名 Controller 时，必须显式指定 Bean 名称，例如 `@RestController("consumerAddressBookController")` 与 `@RestController("agencyAddressBookController")`。
3. **`inner/` 接口安全隔离**：`inner/` 路径下的接口仅供内部 Feign RPC 或模块间信任调用，网关层必须配置全局 Filter 阻断来自外网的直接请求访问 `/inner/**`。
4. **权限校验隔离**：不同端的接口在网关/拦截器层对应不同的 UserType 校验规则（如 `consumer` 端只允许 C 端用户 Token，`operation` 端必须校验管理员权限）。

---

## 3. 数据模型（Domain/DTO/VO）生命周期与流转规则

```text
  [ 前端/调用方 ]
         │
         ▼ (1) 入参: *ReqDTO / *PageQueryReqDTO (带 Validator 校验)
  [ Controller 层 ]
         │
         ▼ (2) 业务入参: *ReqDTO / 组装好的业务实体
  [ Service 业务层 ]
         │
         ├──────────────────────┐
         ▼                      ▼
  [ 转换: BeanUtil ]    [ 核心逻辑处理与状态机流转 ]
         │                      │
         ▼ (3) 实体操作          ▼
  [ Mapper / Entity ]    [ 组装出参: *ResDTO / *VO ]
         │                      │
         ▼ (4) DB 持久化        ▼
  [ MySQL / Redis / ES ]  [ 框架自动包装为 Result<T> / PageResult<T> ]
                                │
                                ▼
                         [ 返回给前端/调用方 ]
```

### 3.1 各模型职责定义与规范
1. **`model.domain.Entity` (持久层实体)**：
   * 对应数据库物理表，必须标注 `@TableName("表名")`，主键标注 `@TableId`。
   * 统一包含基础审计字段：`createTime`、`updateTime`。
   * **严禁规则**：**绝对禁止**将 Entity 直接作为 Controller 的返回值返回给前端；**绝对禁止**将 Entity 作为 Feign API 接口的传输对象。
2. **`model.dto.request.*ReqDTO` (请求数据传输对象)**：
   * 封装客户端或外部调用的请求入参。
   * 必须配合 Hibernate Validator 注解（`@NotNull`, `@NotBlank`, `@Min`, `@Size`）或自定义枚举校验注解（`@EnumValid`），并附带明确的 `message`。
   * 分页查询必须继承 `PageQueryDTO`（包含 `pageNo`, `pageSize`, `sortBy`, `isAsc`）。
3. **`model.dto.response.*ResDTO` (响应数据传输对象)**：
   * 封装返回给前端或微服务 Feign 调用的响应数据。
   * 必须脱敏或剔除密码、内部状态等敏感字段。
4. **`model.vo.*VO` (视图对象)**：
   * 专门用于复杂页面渲染、聚合多个服务数据，或大模型 SSE 流式事件载体（如 `ChatEventVO`）。

### 3.2 对象转换推荐范式
* 单个对象转换：`BeanUtil.toBean(sourceObj, TargetClass.class);`
* 集合批量转换：`BeanUtils.copyToList(sourceList, TargetClass.class);`
* 分页结果转换：`PageUtils.toPage(pageResult, TargetClass.class);`

---

## 4. 分层调用铁律与反模式排查

- **禁止跨层直调**：Controller 严禁直接注入 Mapper 进行数据库操作，所有数据访问必须经过 Service 接口层。
- **禁止参数污染**：已登录接口严禁从请求体获取 `userId`，必须统一从 `UserContext` 获取并在 Service 层注入。
- **禁止在 Service 外部开启事务**：事务必须收敛在 Service 实现类中，Controller 严禁添加 `@Transactional`。
- **SSE / 文件下载接口防误包装**：对于返回 `Flux<T>`、`SseEmitter` 或文件下载的 Controller 方法，必须添加 `@NoWrapper` 注解，避免被全局过滤器包装为 JSON `Result<T>`。