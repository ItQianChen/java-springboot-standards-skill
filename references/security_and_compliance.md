# Spring Boot 安全防护、认证授权与合规审查规范

本规范整合了企业级 Spring Boot 与微服务架构下的安全防御实践，涵盖身份认证、细粒度权限控制、高危漏洞防御（SQL 注入/XSS/CSRF）、敏感数据脱敏、密钥管理与上线前安全审查清单。

---

## 1. 身份认证体系 (Authentication Architecture)

### 1.1 无状态 JWT 认证模型
企业级 RESTful API 服务推荐采用基于 JWT 的无状态认证模式：

```java
package com.company.project.config.security;

import com.company.project.service.JwtService;
import org.springframework.http.HttpHeaders;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * <p>
 * JWT 身份认证过滤器
 * </p>
 *
 * @author Universal Agent
 * @since 1.0.0
 */
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);

        // 1. 校验 Authorization 头格式
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);

            // 2. 验证 Token 有效性并构建 Authentication
            Authentication authentication = jwtService.parseAndAuthenticate(token);
            if (authentication != null) {
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        try {
            filterChain.doFilter(request, response);
        } finally {
            // 3. 严格清理 Security 上下文，防止线程池复用污染
            SecurityContextHolder.clearContext();
        }
    }
}
```

### 1.2 网关鉴权与上下文协同 (Gateway to Service)
* **微服务架构下**：由 API Gateway（如 Spring Cloud Gateway）统一校验 JWT 签名与过期时间。
* 网关鉴权通过后，将解析出的 `userId`, `userType`, `roles` 通过 HTTP Header（如 `X-User-Id`, `X-User-Type`）透传给下游微服务。
* 微服务内部通过 `UserContextInterceptor` 将 Header 变量写入 `UserContext` (ThreadLocal)，**下游微服务无需重复解析 JWT 秘钥**。

### 1.3 Token 撤销与黑名单机制
* JWT 是无状态的，主动登出或踢出登录时，将 `token` 或 `jti` 写入 Redis 黑名单，并设置剩余有效期作为 TTL。
* 鉴权拦截器每次校验时，先检查 Redis 黑名单中是否存在该 Token。

---

## 2. 细粒度授权与访问控制 (Authorization & Access Control)

### 2.1 方法级权限拦截 (`@EnableMethodSecurity`)
在配置类上开启 `@EnableMethodSecurity(prePostEnabled = true)`，在 Controller 或 Service 上使用表达式鉴权：

```java
@RestController
@RequestMapping("/operation/users")
@Api(tags = "用户管理运营端接口")
public class UserOperationController {

    // 角色鉴权
    @PreAuthorize("hasRole('ADMIN')")
    @DeleteMapping("/{id}")
    @ApiOperation("删除用户")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
    }

    // 细粒度权限码鉴权 (自定义 Bean 表达式)
    @PreAuthorize("@perm.hasPerm('user:export')")
    @GetMapping("/export")
    @ApiOperation("导出用户数据")
    public List<UserExportVO> exportUsers() {
        return userService.exportAll();
    }
}
```

### 2.2 六端接口与纵深防御
* **端级物理隔离**：内部服务间接口必须收敛在 `/inner/**` 路径下。
* **网关层防御**：API Gateway 必须配置全局路由拦截规则，**公网请求若访问 `/inner/**` 直接返回 403 Forbidden**，防止微服务内部接口暴露公网。
* **水平越权（IDOR）防御**：操作特定业务资源时，除了角色权限，必须在 Service 层强制比对当前操作人是否拥有该数据的所有权：
  ```java
  // 检查订单所属人是否为当前登录用户
  if (!order.getUserId().equals(UserContext.currentUserId())) {
      throw new ForbiddenOperationException("无权操作他人订单数据");
  }
  ```

---

## 3. 高危安全漏洞防御 (Vulnerability Defenses)

### 3.1 SQL 注入防范 (SQL Injection Defense)
* **MyBatis / MyBatis-Plus 铁律**：
  * **必须使用 `#{}` 占位符**：MyBatis 会将其编译为 JDBC 预编译参数（`?`），有效防范注入。
  * **绝对禁止使用 `${}` 拼接 SQL**：禁止在表名、字段名、排序规则中直接拼接前端未经过滤的字符串。
  * **优先使用 Lambda 链式查询**：使用 MyBatis-Plus 的 `lambdaQuery()`，获得编译期类型安全保障：
    ```java
    // ✅ 正确示范：强类型安全查询
    LambdaQueryWrapper<User> wrapper = Wrappers.<User>lambdaQuery()
        .eq(User::getPhone, phone)
        .eq(User::getStatus, UserStatusEnum.NORMAL.getCode());
    ```
  * **动态排序安全校验**：若业务必须支持动态排序字段，必须使用枚举或白名单校验，杜绝注入：
    ```java
    if (!List.of("create_time", "order_amount", "id").contains(sortBy)) {
        throw new BadRequestException("非法的排序字段");
    }
    ```

### 3.2 XSS 跨站脚本防御与入参清洗
* **Bean Validation 约束**：所有 DTO 字段使用 `@NotBlank`, `@Size(max = ...)`, `@Pattern` 进行严格格式与长度限制。
* **富文本防 XSS**：若接口允许接收 HTML 富文本（如文章正文、商品详情），必须在存库或渲染前通过 HTML 白名单清理工具（如 Jsoup）清洗标签：
  ```java
  public static String cleanHtml(String unsafeHtml) {
      if (StringUtils.isBlank(unsafeHtml)) {
          return unsafeHtml;
      }
      return Jsoup.clean(unsafeHtml, Safelist.relaxed());
  }
  ```

### 3.3 CSRF 跨站请求伪造策略
* **纯 RESTful API (Bearer Token 体系)**：因客户端不依赖 Cookie 自动携带凭据，可安全禁用 CSRF：
  ```java
  http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
  ```
* **基于 Cookie 的 Web 应用**：必须启用 CSRF Protection，并在 Cookie 中添加 `HttpOnly`、`Secure`、`SameSite=Strict` 属性。

---

## 4. 密钥管理与敏感数据脱敏 (Secrets & PII Protection)

### 4.1 密钥与凭据管理
* **严禁代码硬编码**：禁止在任何 Java 代码、XML 或 `application.yml` 中写入明文数据库密码、云厂商 AccessKey/SecretKey、支付私钥。
* **环境与配置中心隔离**：生产环境密钥通过系统环境变量（Environment Variables）或加密配置中心（Nacos Config 加加密插件 / HashiCorp Vault）动态注入。

### 4.2 日志脱敏与敏感信息防泄漏 (PII Masking)
* **禁止打印的内容**：禁止在日志中打印密码原文、完整 Token、加密私钥、完整银行卡号（PAN）。
* **脱敏规则标准**：
  * 手机号：保留前 3 后 4，中间 4 位脱敏（`138****1234`）
  * 身份证号：保留前 6 后 4，中间脱敏（`110101********1234`）
  * 邮箱：前缀保留前 1 位，其余脱敏（`z***@example.com`）
* **脱敏实现**：利用 Hutool `DesensitizedUtil` 或自定义 Jackson Serializer / Logback 脱敏插件自动转换。

---

## 5. 安全响应头与限流防御 (Headers & Rate Limiting)

### 5.1 Spring Security 安全头配置
生产环境必须配置标准安全响应头：

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
);
```

### 5.2 接口防刷与速率限制 (Rate Limiting)
对于登录、验证码发送、支付下单、AI 对话生成等高开销端点，必须配置限流策略：
* **单机/轻量**：基于 Bucket4j 令牌桶算法。
* **集群/微服务**：基于 Redis + Lua 令牌桶或 Spring Cloud Gateway RequestRateLimiter，触发限流时返回 HTTP 429 (`Too Many Requests`)。

---

## 6. 文件上传与存储安全 (File Upload Security)

1. **后缀与 MIME 双重校验**：不仅校验文件扩展名白名单（如 `.jpg`, `.png`, `.pdf`），还必须通过 Apache Tika 校验文件的魔数与实际 MIME-Type。
2. **文件大小限制**：在配置中设置单文件与总请求体上限（`spring.servlet.multipart.max-file-size=10MB`）。
3. **存储隔离与重命名**：
   * 严禁将用户上传的文件直接存放在 Web 应用根目录。
   * 上传文件必须使用随机 UUID 重命名（如 `uuid.jpg`），消除目录穿越（Path Traversal）漏洞。
   * 优先转存至阿里云 OSS / 腾讯云 COS / 华为云 OBS / 本地 MinIO，并设置防盗链与只读权限。

---

## 7. 发布前安全审查清单 (Pre-Release Security Checklist)

在任何功能上线或合并前，必须逐项核对：

- [ ] **认证有效性**：所有需登录接口是否校验了 Token？Token 是否有合理过期时间与续期机制？
- [ ] **权限守卫覆盖**：敏感业务操作是否添加了 `@PreAuthorize` 权限控制？
- [ ] **横向越权防御**：操作订单、用户资料、账单等数据时，是否校验了数据属于当前 `UserContext.currentUserId()`？
- [ ] **SQL 注入扫描**：所有 Mapper XML 和 Java 代码中，是否完全杜绝了 `${}` 拼接 SQL？
- [ ] **内部接口隔离**：`/inner/**` 接口是否确认已被网关层拦截，外部网络无法直连？
- [ ] **敏感数据脱敏**：出参 DTO/VO 是否去除了密码、Salt 等字段？日志是否杜绝了明文身份证/手机号打印？
- [ ] **密钥无泄漏**：提交的代码和配置文件中没有任何明文密码、AK/SK、Token？
- [ ] **文件上传安全**：上传接口是否限制了格式白名单与文件大小？是否使用 UUID 重命名？
- [ ] **依赖漏洞检查**：CI/CD 流程中是否执行了依赖扫描（如 OWASP Dependency-Check），无严重 CVE 风险？
