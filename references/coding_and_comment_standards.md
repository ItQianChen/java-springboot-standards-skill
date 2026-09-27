# 编码规范、统一异常、响应体系与注释标准

本规范定义了 Spring Boot 工程中的命名规范、用户上下文访问、统一响应封装、全局异常与 600+ 业务错误码体系，以及 Class Javadoc、方法文档、业务步骤分步注释与 SSE 流式接口的标准范式。

---

## 1. 核心命名与代码风格规范

| 元素 | 命名规范 | 示例 | 禁忌 |
| :--- | :--- | :--- | :--- |
| **包名** | 全小写单数，用点分隔 | `com.jzo2o.customer.service.impl` | 严禁大写或下划线 |
| **接口与实现类** | 接口 `I*Service`，实现类 `*ServiceImpl` | `IOrdersService` / `OrdersServiceImpl` | 严禁省略 `I` 或随意拼写 |
| **Mapper 接口** | `*Mapper` | `OrdersMapper` | 严禁命名为 `*Dao` (保持项目一致) |
| **请求入参 DTO** | `*ReqDTO` / `*PageQueryReqDTO` | `PlaceOrderReqDTO`, `OrderPageQueryReqDTO` | 严禁入参直接使用 Entity |
| **响应出参 DTO** | `*ResDTO` | `OrderResDTO`, `AddressBookResDTO` | 严禁直接向前端返回 Entity |
| **视图对象** | `*VO` | `AddressBookVO`, `ChatEventVO` | 仅在需要高度定制展示视图或 SSE 流式事件时使用 |
| **常量类与字段** | 集中在 `constants` 包，全大写下划线 | `RedisConstants.CacheName.SERVE` | 严禁在业务代码中硬编码魔法值 |
| **枚举类** | 类名 `*Enum`，枚举项全大写下划线 | `OrderStatusEnum.NO_PAY` | 必须提供 `status/code` 与 `desc` 字段 |
| **AI 工具类** | `*Tools`，方法使用 `@Tool` 声明 | `CourseTools.queryCourseById` | 必须提供明确的英文/中文描述供模型理解 |

---

## 2. 用户上下文与安全访问规范 (`UserContext`)

在已鉴权接口中，当前登录用户的主体信息由网关解析并在拦截器层注入 `UserContext` 线程局部变量（ThreadLocal）。

### 2.0 上下文定义、填充与清理

```java
public final class UserContext {

    private static final ThreadLocal<CurrentUserInfo> CONTEXT = new ThreadLocal<>();

    private UserContext() {
        // 工具类禁止实例化
    }

    public static void setUser(CurrentUserInfo user) {
        CONTEXT.set(user);
    }

    public static CurrentUserInfo currentUser() {
        return CONTEXT.get();
    }

    public static Long currentUserId() {
        CurrentUserInfo user = currentUser();
        return user == null ? null : user.getId();
    }

    public static Integer getUserType() {
        CurrentUserInfo user = currentUser();
        return user == null ? null : user.getUserType();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

填充与清理必须成对出现在同一条 MVC 拦截链中。单体应用可在 `preHandle` 解析 Session/JWT 并写入；网关后的微服务从 `X-User-Id`、`X-User-Type` 等可信 Header 构建：

```java
@Component
public class UserContextInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        CurrentUserInfo user = parseTrustedUser(request);
        UserContext.setUser(user);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        // Tomcat 复用工作线程；不 remove 会把上一个请求的用户带给下一个请求
        UserContext.clear();
    }
}
```

生命周期约束：

1. `setUser` 只能由鉴权 Filter/Interceptor 或框架上下文组件调用，业务 Controller/Service 禁止写入或覆盖。
2. 清理必须放在 `afterCompletion` 或 `finally`，保证正常返回、业务异常、鉴权拦截后都执行。
3. 异步线程、`@Async`、线程池、CompletableFuture 不自动继承 ThreadLocal；进入异步逻辑前显式复制用户快照，任务结束后同步清理。
4. WebFlux/Reactor 不得使用该 ThreadLocal 模式，应使用 Reactor Context 或框架安全上下文。
5. 定时任务、MQ 消费者、系统任务没有请求上下文时，使用明确系统身份，不得伪造用户身份。

### 2.1 规范要求
1. **绝对禁止**在 `@RequestBody`、`@RequestParam` 或 `@PathVariable` 中接收当前用户的 `userId`。
2. 业务代码统一通过 `UserContext` 安全获取当前登录人属性：

```java
// 获取当前登录用户 ID (Long)
Long currentUserId = UserContext.currentUserId();

// 获取当前登录用户完整对象 (包含 userId, userType, name 等)
CurrentUserInfo currentUser = UserContext.currentUser();

// 获取用户身份类型 (1-普通客户, 2-服务人员, 3-机构人员, 4-平台运营人员)
Integer userType = UserContext.getUserType();
```

---

## 3. 统一响应、分页与 SSE 流式接口规范

### 3.1 Controller 响应自包装机制
基于 MVC 框架提供的全局 Filter / Advice 包装机制，Controller 层方法**直接返回具体业务 DTO、VO、集合或 `void`**，框架层将自动封装为标准的 `Result<T>`（`code: 200, msg: "OK", data: ...`）。

```java
// 正确范式：直接返回业务对象或 void
@PostMapping("/add")
@ApiOperation("地址薄新增")
public void add(@RequestBody AddressBookUpsertReqDTO reqDTO) {
    addressBookService.add(reqDTO);
}

@GetMapping("/{id}")
@ApiOperation("地址薄详情")
public AddressBookResDTO detail(@PathVariable("id") Long id) {
    return addressBookService.findById(id);
}

// 错误反模式：严禁在 Controller 中手动 new Result<>()
// public Result<AddressBookResDTO> detail(...) { return Result.ok(res); } // 严禁！会导致双层 Result 嵌套
```

### 3.2 SSE 流式接口与 `@NoWrapper` 穿透规范
对于返回 `Flux<T>`、`SseEmitter` 或文件下载的 Controller 方法，**必须添加 `@NoWrapper` 注解**，防止全局过滤器将其强行拦截包装为 JSON `Result<T>`，从而破坏流式传输通道：

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/chat")
@Api(tags = "AI 智能对话与流式问答接口")
public class ChatController {
    private final ChatService chatService;

    @NoWrapper // 核心：跳过统一 Result 包装，直通 SSE 文本流
    @PostMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    @ApiOperation("流式智能对话接口 (SSE)")
    public Flux<ChatEventVO> chat(@RequestBody ChatDTO dto) {
        return chatService.chat(dto);
    }
}
```

### 3.3 分页查询标准范式
1. 分页入参 DTO 必须继承 `PageQueryDTO`。
2. Controller 返回 `PageResult<T>`。
3. Service 层通过 `PageUtils` 或 `PageHelperUtils` 进行分页执行与对象投影转换：

```java
// Controller
@GetMapping("/page")
@ApiOperation("订单分页查询")
public PageResult<OrderResDTO> page(OrderPageQueryReqDTO queryReqDTO) {
    return ordersService.page(queryReqDTO);
}

// Service 实现类
@Override
public PageResult<OrderResDTO> page(OrderPageQueryReqDTO queryReqDTO) {
    // 1. 构建 MyBatis-Plus 分页对象
    Page<Orders> page = PageUtils.parsePageQuery(queryReqDTO, Orders.class);

    // 2. 构造查询条件
    LambdaQueryWrapper<Orders> queryWrapper = Wrappers.<Orders>lambdaQuery()
            .eq(Orders::getUserId, UserContext.currentUserId())
            .orderByDesc(Orders::getCreateTime);

    // 3. 执行分页查询并转换为出参 DTO
    Page<Orders> ordersPage = baseMapper.selectPage(page, queryWrapper);
    return PageUtils.toPage(ordersPage, OrderResDTO.class);
}
```

---

## 4. 统一异常体系与 600+ 业务错误码规范

系统采用基于 HTTP 状态与领域错误码结合的分层异常体系，由全局异常拦截器（`CommonExceptionAdvice`）统一处理并转换为友好提示。

### 4.1 核心异常类及使用场景
1. **`BadRequestException` (HTTP 400)**：客户端请求参数不合法、校验失败、数据格式错误。
   ```java
   if (serveResDTO == null || serveResDTO.getSaleStatus() != 2) {
       throw new BadRequestException("服务不可用或已下架");
   }
   ```
2. **`ForbiddenOperationException` (HTTP 403)**：当前状态禁止此操作、业务规则前置校验不满足。
   ```java
   if (order.getOrdersStatus() != OrderStatusEnum.NO_PAY.getStatus()) {
       throw new ForbiddenOperationException("只有待支付订单方可执行取消操作");
   }
   ```
3. **`CommonException(code, msg)` (业务领域失败)**：涉及明确的业务处理失败、第三方调用失败、高并发抢占失败等，携带明确的业务错误码。
   ```java
   throw new CommonException(ErrorInfo.Code.SEIZE_ORDERS_FAILD, "抢单失败，该订单已被其他师傅接单");
   ```

### 4.2 业务错误码定义规范 (`ErrorInfo.Code`)
错误码必须统一维护在 `constants.ErrorInfo` 中，按业务段严格分配：

```java
public class ErrorInfo {
    public static class Code {
        public static final int NOT_LOGIN = 600;                // 未登录
        public static final int LOGIN_TIMEOUT = 601;            // 登录过期
        public static final int ILLEGAL_TOKEN = 602;            // token不合法
        public static final int NO_PERMISSIONS = 603;           // 无权限访问
        public static final int FORBIDDEN_OPERATION = 604;      // 禁止操作
        public static final int ACCOUNT_FREEZED = 605;          // 账号冻结
        public static final int DISPATCH_REJECT = 606;          // 派单拒单
        public static final int ORDERS_CANCEL = 607;            // 取消订单失败
        public static final int SEIZE_ORDERS_FAILD = 608;       // 抢单失败
        public static final int TRADE_FAILED = 609;             // 交易/支付失败
        public static final int HTTP_EVALUATION_FAILED = 610;   // 评价系统调用失败
        public static final int SEIZE_COUPON_FAILD = 611;       // 抢券失败
    }
}
```

---

## 5. Javadoc 与业务步骤注释规范

代码注释是系统可维护性的基石。所有代码必须具备清晰的 Javadoc 契约说明与结构化的方法内业务逻辑注释。

### 5.1 类级 Javadoc 注释规范
所有 Class、Interface 头部必须具备标准化 Javadoc，说明该类的职责与归属：
```java
/**
 * <p>
 * 订单核心创建与支付流转服务实现类
 * </p>
 *
 * @author itcast
 * @since 2023-07-10
 */
@Slf4j
@Service
public class OrdersCreateServiceImpl extends ServiceImpl<OrdersMapper, Orders> implements IOrdersCreateService {
    // ...
}
```

### 5.2 方法级 Javadoc 规范
非自解释方法、Service 接口及公共方法必须包含业务目的、所有参数含义与返回值说明：
```java
/**
 * 用户下单并生成预支付订单
 *
 * @param placeOrderReqDTO 下单请求体参数 (包含服务项ID、预约时间、地址ID等)
 * @return 下单结果DTO (包含订单ID与支付状态)
 * @throws BadRequestException 当地址不存在或服务已下架时抛出
 * @throws ForbiddenOperationException 当用户存在未结清订单时抛出
 */
@Override
public PlaceOrderResDTO placeOrder(PlaceOrderReqDTO placeOrderReqDTO) {
    // ...
}
```

### 5.3 方法内部序号化步骤注释与 Why 规范
**强制规则**：业务方法内部必须使用 `// 1. xxx`、`// 2. xxx` 的序号格式拆解执行步骤；对非显然的业务设计决策、阈值设置、并发保护必须附带 **Why（为什么这样做）** 说明。

```java
@Override
@Transactional(rollbackFor = Exception.class)
public PlaceOrderResDTO placeOrder(PlaceOrderReqDTO placeOrderReqDTO) {
    // 1. 数据合法性校验与前置检查
    AddressBookResDTO addressDetail = addressBookApi.detail(placeOrderReqDTO.getAddressBookId());
    if (addressDetail == null) {
        throw new BadRequestException("预约服务地址不存在，无法下单");
    }

    ServeAggregationResDTO serveDTO = serveApi.findById(placeOrderReqDTO.getServeId());
    // 只有上架状态 (saleStatus == 2) 的服务允许下单
    if (serveDTO == null || serveDTO.getSaleStatus() != 2) {
        throw new BadRequestException("服务不可用或已下架");
    }

    // 2. 下单数据准备与订单快照装配
    Orders orders = new Orders();
    // 订单ID格式: {yyMMdd}{13位Redis分布式序列号}，保证全局唯一且按时间有序
    orders.setId(generatorOrderId());
    orders.setUserId(UserContext.currentUserId());
    orders.setOrdersStatus(OrderStatusEnum.NO_PAY.getStatus());
    orders.setPayStatus(OrderPayStatusEnum.NO_PAY.getStatus());
    orders.setServeStartTime(placeOrderReqDTO.getServeStartTime());
    orders.setPurNum(NumberUtils.null2Default(placeOrderReqDTO.getPurNum(), 1));

    // 冗余下单时刻的服务项名称、快照图片与地址，避免后续基础数据变更影响历史订单
    orders.setServeItemName(serveDTO.getServeItemName());
    orders.setServeItemImg(serveDTO.getServeItemImg());
    orders.setTotalAmount(serveDTO.getPrice().multiply(new BigDecimal(orders.getPurNum())));
    orders.setDiscountAmount(BigDecimal.ZERO);
    orders.setRealPayAmount(orders.getTotalAmount());

    // 3. 执行持久化与事务分支
    if (ObjectUtils.isNotNull(placeOrderReqDTO.getCouponId())) {
        // 使用优惠券下单时，调用远程市场服务核销卡券，必须走 Seata 全局分布式事务
        owner.addWithCoupon(orders, placeOrderReqDTO.getCouponId());
    } else {
        // 普通下单走本地 Spring 事务
        owner.add(orders);
    }

    // 4. 驱动状态机启动
    OrderSnapshotDTO snapshot = BeanUtil.toBean(orders, OrderSnapshotDTO.class);
    orderStateMachine.start(orders.getUserId(), String.valueOf(orders.getId()), snapshot);

    return new PlaceOrderResDTO(orders.getId());
}
```

---

## 6. Knife4j / Swagger 接口文档注解规范

所有对外暴露的 Controller 及其 DTO 字段必须完整标注文档注解，确保接口文档自解释：

```java
// Controller 注解规范
@RestController("consumerAddressBookController")
@RequestMapping("/consumer/address-book")
@Api(tags = "用户端 - 地址薄相关接口")
public class AddressBookController {

    @PutMapping("/{id}")
    @ApiOperation("地址薄修改")
    @ApiImplicitParams({
        @ApiImplicitParam(name = "id", value = "地址薄ID", required = true, dataTypeClass = Long.class)
    })
    public void update(@NotNull(message = "ID不能为空") @PathVariable("id") Long id,
                       @Validated @RequestBody AddressBookUpsertReqDTO reqDTO) {
        addressBookService.update(id, reqDTO);
    }
}

// DTO 注解规范
@Data
@ApiModel(description = "地址薄新增或修改请求体")
public class AddressBookUpsertReqDTO {

    @ApiModelProperty(value = "详细地址", required = true, example = "北京市朝阳区三里屯路1号")
    @NotBlank(message = "详细地址不能为空")
    private String address;

    @ApiModelProperty(value = "经纬度坐标 (格式: 经度,纬度)", example = "116.45528,39.93774")
    private String location;

    @ApiModelProperty(value = "是否默认地址 (0-否, 1-是)", required = true, example = "1")
    @NotNull(message = "默认状态不能为空")
    @EnumValid(enumeration = {0, 1}, message = "默认状态值必须为0或1")
    private Integer isDefault;
}
```
