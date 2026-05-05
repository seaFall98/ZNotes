

## 结论

**JWT 适合放在网关层做“统一认证”，但不适合让网关承担复杂业务授权。**

推荐模型是：

```text
客户端携带 JWT
        ↓
API Gateway 校验 JWT
        ↓
Gateway 提取用户身份
        ↓
Gateway 注入可信 Header
        ↓
下游微服务基于 Header 做业务授权
```

也就是：

```text
网关负责：你是谁？Token 是否可信？
服务负责：你能不能操作这个业务资源？
```

---

# 1. JWT 本质是什么？

JWT，全称是 **JSON Web Token**。

它本质上是一个经过签名的字符串，用来表达一组可信声明。

典型 JWT 长这样：

```text
xxxxx.yyyyy.zzzzz
```

它由三段组成：

```text
Header.Payload.Signature
```

例如：

```json
Header:
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload:
{
  "sub": "10001",
  "username": "zhangsan",
  "roles": ["USER", "ADMIN"],
  "tenantId": "tenant-a",
  "iat": 1710000000,
  "exp": 1710003600
}

Signature:
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

核心理解：

> **JWT 的 Payload 不是加密的，而是 Base64Url 编码的。任何拿到 Token 的人都可以解码看到 Payload 内容。真正保证可信的是 Signature。**

所以 JWT 里不要放：

```text
密码
身份证号
手机号
银行卡
敏感业务数据
```

应该只放：

```text
userId
tenantId
roles
authorities
tokenType
issuedAt
expiredAt
```

---

# 2. JWT 解决了什么问题？

传统 Session 模型：

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant R as Redis / Session Store
    participant G as Gateway
    participant S as Service

    C->>A: 登录 username/password
    A->>R: 保存 sessionId -> userInfo
    A-->>C: 返回 sessionId
    C->>G: 请求携带 sessionId
    G->>R: 查询 session
    R-->>G: 返回 userInfo
    G->>S: 转发请求
```

JWT 模型：

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant G as Gateway
    participant S as Service

    C->>A: 登录 username/password
    A-->>C: 返回 JWT
    C->>G: 请求携带 JWT
    G->>G: 本地验签 + 校验 exp
    G->>S: 透传 userId / tenantId / roles
```

JWT 的核心优势：

```text
网关可以本地验签
不必每次查 Redis / DB
天然适合微服务横向扩展
适合无状态认证
```

但 JWT 的核心问题也很明显：

```text
Token 一旦签发，在过期前天然有效
服务端很难主动让它失效
```

所以生产上通常不是“只用 JWT”，而是：

```text
短期 Access Token
+ 长期 Refresh Token
+ 黑名单 / Token Version / Redis
```

---

# 3. Access Token 和 Refresh Token

企业项目中，推荐拆成两类 Token。

|Token|用途|生命周期|是否频繁发送|
|---|---|--:|---|
|Access Token|调接口|短，例如 15 分钟 / 30 分钟 / 2 小时|是|
|Refresh Token|换新 Access Token|长，例如 7 天 / 30 天|否|

流程：

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant G as Gateway
    participant S as Service

    C->>A: 登录
    A-->>C: accessToken + refreshToken

    C->>G: 请求业务接口，携带 accessToken
    G->>G: 校验 accessToken
    G->>S: 转发业务请求

    C->>A: accessToken 过期，用 refreshToken 换新
    A->>A: 校验 refreshToken
    A-->>C: 新 accessToken
```

推荐策略：

```text
Access Token：15 ~ 60 分钟
Refresh Token：7 ~ 30 天
Refresh Token 存 Redis / DB，可主动吊销
```

原因：

- Access Token 短，泄露后的风险窗口小
    
- Refresh Token 可控，可以服务端主动失效
    
- 用户退出登录时，只需要删除 Refresh Token 或加入黑名单
    
- 密码修改、账号冻结时，可以让 Refresh Token 失效
    

---

# 4. JWT 应该放在哪里？

常见有两种方式。

## 4.1 放在 Authorization Header

推荐用于 App、后端服务、前后端分离 API：

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

优点：

```text
语义清晰
不容易被浏览器自动携带
适合移动端、桌面端、API 调用
```

缺点：

```text
前端通常要存 localStorage / memory
要注意 XSS 风险
```

## 4.2 放在 Cookie

推荐用于传统 Web 或 BFF 模式：

```http
Cookie: access_token=xxx
```

优点：

```text
可以设置 HttpOnly
可以减少 JS 读取 Token 的风险
```

缺点：

```text
浏览器会自动携带 Cookie
需要额外防 CSRF
跨域配置更复杂
```

实践建议：

```text
纯 API / App：
  Authorization Bearer Token

浏览器 Web：
  HttpOnly + Secure + SameSite Cookie
  同时做好 CSRF 防护

前后端分离但自己掌控前端：
  可以 Authorization Header，但要严格防 XSS
```

---

# 5. Gateway 中 JWT 认证的标准流程

```mermaid
flowchart TD
    A[Client Request] --> B[Gateway GlobalFilter]
    B --> C{是否白名单路径}
    C -->|是| D[直接放行]
    C -->|否| E{是否有 Authorization Header}
    E -->|否| F[401 Unauthorized]
    E -->|是| G{Bearer 格式是否正确}
    G -->|否| F
    G -->|是| H[解析 JWT]
    H --> I{签名是否正确}
    I -->|否| F
    I -->|是| J{是否过期}
    J -->|是| F
    J -->|否| K{issuer/audience 是否正确}
    K -->|否| F
    K -->|是| L[提取 userId tenantId roles]
    L --> M[注入内部 Header]
    M --> N[转发下游服务]
```

网关至少要校验：

|校验项|说明|
|---|---|
|signature|签名是否合法|
|exp|是否过期|
|nbf|是否尚未生效|
|iss|签发方是否可信|
|aud|接收方是否正确|
|token_type|是否是 access token|
|jti|Token 唯一 ID，可用于黑名单|
|alg|算法是否符合预期|

重点：

> 不要只解析 Payload。必须验签。

错误做法：

```java
// 错误：只 decode，不 verify
JWT.decode(token).getSubject();
```

正确做法：

```java
// 正确：verify 后再取 claims
JWTVerifier verifier = JWT.require(algorithm)
        .withIssuer("devwiki-auth")
        .withAudience("devwiki-api")
        .build();

DecodedJWT jwt = verifier.verify(token);
```

---

# 6. Spring Cloud Gateway 示例：JWT GlobalFilter

下面给一个更接近生产结构的示例。

## 6.1 JWT 配置类

```java
@ConfigurationProperties(prefix = "security.jwt")
public record JwtProperties(
        String issuer,
        String audience,
        String secret,
        List<String> whitelist
) {
}
```

配置：

```yaml
security:
  jwt:
    issuer: devwiki-auth
    audience: devwiki-api
    secret: "replace-with-at-least-256-bit-secret"
    whitelist:
      - /api/auth/login
      - /api/auth/register
      - /actuator/health
```

---

## 6.2 JWT 服务

以 `java-jwt` 为例：

```java
@Component
@RequiredArgsConstructor
public class JwtVerifierService {

    private final JwtProperties properties;

    private JWTVerifier verifier() {
        Algorithm algorithm = Algorithm.HMAC256(properties.secret());

        return JWT.require(algorithm)
                .withIssuer(properties.issuer())
                .withAudience(properties.audience())
                .acceptLeeway(5) // 允许 5 秒时钟偏移
                .build();
    }

    public AuthenticatedUser verifyAccessToken(String token) {
        DecodedJWT jwt = verifier().verify(token);

        String tokenType = jwt.getClaim("token_type").asString();
        if (!"access".equals(tokenType)) {
            throw new JWTVerificationException("Invalid token type");
        }

        String userId = jwt.getSubject();
        String tenantId = jwt.getClaim("tenant_id").asString();
        List<String> roles = jwt.getClaim("roles").asList(String.class);

        if (userId == null || userId.isBlank()) {
            throw new JWTVerificationException("Missing subject");
        }

        return new AuthenticatedUser(userId, tenantId, roles);
    }
}
```

用户对象：

```java
public record AuthenticatedUser(
        String userId,
        String tenantId,
        List<String> roles
) {
}
```

---

## 6.3 Gateway GlobalFilter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationGlobalFilter implements GlobalFilter, Ordered {

    private final JwtProperties jwtProperties;
    private final JwtVerifierService jwtVerifierService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 1. 白名单路径放行
        if (isWhitelisted(path)) {
            return chain.filter(exchange);
        }

        // 2. 提取 Authorization Header
        String authorization = exchange.getRequest()
                .getHeaders()
                .getFirst(HttpHeaders.AUTHORIZATION);

        if (authorization == null || !authorization.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }

        String token = authorization.substring("Bearer ".length());

        try {
            // 3. 验签 + 校验 exp/iss/aud/token_type
            AuthenticatedUser user = jwtVerifierService.verifyAccessToken(token);

            // 4. 删除外部伪造的内部身份 Header
            ServerHttpRequest cleanRequest = exchange.getRequest()
                    .mutate()
                    .headers(headers -> {
                        headers.remove("X-User-Id");
                        headers.remove("X-Tenant-Id");
                        headers.remove("X-User-Roles");
                    })
                    .build();

            // 5. 由网关注入可信 Header
            ServerHttpRequest mutatedRequest = cleanRequest.mutate()
                    .header("X-User-Id", user.userId())
                    .header("X-Tenant-Id", nullToEmpty(user.tenantId()))
                    .header("X-User-Roles", String.join(",", user.roles()))
                    .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JWTVerificationException ex) {
            return unauthorized(exchange, "Invalid token");
        }
    }

    private boolean isWhitelisted(String path) {
        return jwtProperties.whitelist()
                .stream()
                .anyMatch(path::startsWith);
    }

    private String nullToEmpty(String value) {
        return value == null ? "" : value;
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = """
                {
                  "code": "UNAUTHORIZED",
                  "message": "%s"
                }
                """.formatted(message);

        DataBuffer buffer = response.bufferFactory()
                .wrap(body.getBytes(StandardCharsets.UTF_8));

        return response.writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

重点是这一步：

```java
headers.remove("X-User-Id");
headers.remove("X-Tenant-Id");
headers.remove("X-User-Roles");
```

因为客户端可能伪造：

```http
X-User-Id: 1
X-User-Roles: ADMIN
```

网关必须先清理，再重新注入。

---

# 7. 下游服务如何接收用户身份？

在下游服务里，不建议每个 Controller 都手动读取 Header。

可以用拦截器统一处理。

## 7.1 用户上下文

```java
public record UserContext(
        String userId,
        String tenantId,
        List<String> roles
) {
}
```

```java
public final class UserContextHolder {

    private static final ThreadLocal<UserContext> HOLDER = new ThreadLocal<>();

    private UserContextHolder() {
    }

    public static void set(UserContext context) {
        HOLDER.set(context);
    }

    public static UserContext get() {
        UserContext context = HOLDER.get();
        if (context == null) {
            throw new IllegalStateException("Missing user context");
        }
        return context;
    }

    public static void clear() {
        HOLDER.remove();
    }
}
```

## 7.2 Spring MVC 拦截器

```java
@Component
public class UserContextInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler
    ) {
        String userId = request.getHeader("X-User-Id");
        String tenantId = request.getHeader("X-Tenant-Id");
        String rolesHeader = request.getHeader("X-User-Roles");

        if (userId == null || userId.isBlank()) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "Missing user context");
        }

        List<String> roles = rolesHeader == null || rolesHeader.isBlank()
                ? List.of()
                : Arrays.asList(rolesHeader.split(","));

        UserContextHolder.set(new UserContext(userId, tenantId, roles));
        return true;
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex
    ) {
        UserContextHolder.clear();
    }
}
```

注册：

```java
@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final UserContextInterceptor userContextInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(userContextInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns("/actuator/health");
    }
}
```

业务代码里使用：

```java
@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderRepository orderRepository;

    public OrderDetail getOrderDetail(Long orderId) {
        UserContext user = UserContextHolder.get();

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new NotFoundException("Order not found"));

        // 业务授权：只能查看自己的订单
        if (!order.getUserId().equals(user.userId())) {
            throw new AccessDeniedException("No permission to access this order");
        }

        return OrderDetail.from(order);
    }
}
```

这就是：

```text
网关认证
服务授权
```

---

# 8. JWT 与 RBAC / ABAC

JWT 里可以放角色和权限，但不要放太多。

## 8.1 RBAC

RBAC 是基于角色的访问控制：

```text
USER
ADMIN
SUPER_ADMIN
```

JWT Payload：

```json
{
  "sub": "10001",
  "roles": ["ADMIN"]
}
```

适合判断：

```text
是否能访问后台管理接口
是否能调用管理员 API
是否能进入某个模块
```

例如网关可以简单做：

```text
/api/admin/** 需要 ADMIN
```

但不要在网关判断复杂业务权限。

---

## 8.2 ABAC

ABAC 是基于属性的访问控制：

```text
用户属性
资源属性
环境属性
租户属性
```

例如：

```text
只有订单 owner 可以查看订单
只有同租户用户可以访问租户数据
只有项目成员可以编辑项目文档
```

这种授权必须在服务内部做，因为它通常需要查业务库。

例如 DevWiki 项目：

```text
用户 A 是否能编辑 wikiId=123？
需要查：
  wiki.ownerId
  wiki.teamId
  user.teamRole
  wiki.visibility
```

这不应该放网关。

---

# 9. Token 吊销怎么做？

JWT 最大的问题是：**签发后无法天然撤销。**

常用方案有三种。

---

## 9.1 短 Access Token + Refresh Token

这是首选。

```text
Access Token 15 分钟过期
Refresh Token 存 Redis / DB
退出登录删除 Refresh Token
```

优点：

```text
简单
性能好
安全性足够
```

缺点：

```text
Access Token 在剩余有效期内仍可使用
```

一般可以接受。

---

## 9.2 JWT 黑名单

JWT 里加入 `jti`：

```json
{
  "sub": "10001",
  "jti": "token-uuid-001",
  "exp": 1710003600
}
```

退出登录时：

```text
Redis:
blacklist:jti:token-uuid-001 = true
TTL = token 剩余过期时间
```

网关每次验证 JWT 后，再查 Redis：

```java
Boolean blacklisted = redisTemplate.hasKey("blacklist:jti:" + jti);
if (Boolean.TRUE.equals(blacklisted)) {
    throw new JWTVerificationException("Token revoked");
}
```

优点：

```text
可以主动吊销 Access Token
```

缺点：

```text
每次请求多一次 Redis 查询
JWT 的无状态优势被削弱
```

适合安全要求较高的后台系统。

---

## 9.3 Token Version

用户表中维护：

```text
user.token_version = 3
```

JWT Payload：

```json
{
  "sub": "10001",
  "token_version": 3
}
```

当用户修改密码、管理员冻结账号时：

```text
user.token_version++
```

网关或 Auth Service 校验：

```text
JWT.token_version == current_user.token_version
```

优点：

```text
可以批量失效某个用户所有 Token
```

缺点：

```text
需要查 DB / Redis
```

实践中可以把用户状态缓存到 Redis：

```text
auth:user:10001:token_version = 3
auth:user:10001:status = ENABLED
```

---

# 10. JWT 签名算法怎么选？

常见算法：

|算法|类型|特点|
|---|---|---|
|HS256|对称签名|简单，签发和验证用同一个 secret|
|RS256|非对称签名|私钥签发，公钥验证|
|ES256|非对称签名|更短签名，性能和兼容性需评估|

## 小型系统

可以用：

```text
HS256
```

但要注意：

```text
secret 足够长
不能提交到 Git
不同环境不同 secret
定期轮换
```

## 中大型系统

推荐：

```text
RS256
```

架构：

```text
Auth Service 持有 private key
Gateway 持有 public key
下游服务可选持有 public key
```

好处：

```text
只有 Auth Service 能签发 Token
Gateway 只能验证，不能伪造 Token
密钥职责更清晰
```

微服务环境更推荐 RS256。

---

# 11. JWKS：企业级密钥发布方式

如果使用 RS256，常见做法是 Auth Service 暴露 JWKS：

```text
/.well-known/jwks.json
```

里面包含公钥集合。

JWT Header 里带：

```json
{
  "alg": "RS256",
  "kid": "key-2026-05"
}
```

Gateway 根据 `kid` 找对应公钥验签。

这样可以做密钥轮换：

```text
新私钥签发新 Token
旧公钥继续验证旧 Token
等旧 Token 全部过期后删除旧公钥
```

这比直接在网关配置一个固定 public key 更灵活。

---

# 12. 多租户系统里的 JWT

如果系统支持多租户，JWT 至少要包含：

```json
{
  "sub": "10001",
  "tenant_id": "tenant-a",
  "roles": ["TENANT_ADMIN"]
}
```

网关注入：

```http
X-User-Id: 10001
X-Tenant-Id: tenant-a
X-User-Roles: TENANT_ADMIN
```

下游服务必须用 `tenant_id` 做数据隔离。

例如查询订单时：

```sql
SELECT *
FROM orders
WHERE id = ?
  AND tenant_id = ?
```

不要只写：

```sql
SELECT *
FROM orders
WHERE id = ?
```

这是多租户系统高危漏洞。

---

# 13. JWT 常见安全坑

## 13.1 只 Decode，不 Verify

错误：

```java
DecodedJWT jwt = JWT.decode(token);
String userId = jwt.getSubject();
```

这只是解码，没有验签。

攻击者可以伪造：

```json
{
  "sub": "1",
  "roles": ["ADMIN"]
}
```

必须用 `verifier.verify(token)`。

---

## 13.2 信任客户端传来的 X-User-Id

错误：

```http
GET /api/order/1
X-User-Id: 1
```

如果下游服务信任这个 Header，而服务又能被外部直接访问，就完了。

正确做法：

```text
外部只能访问 Gateway
Gateway 清理外部 X-User-Id
Gateway 重新注入 X-User-Id
下游只允许网关访问
```

---

## 13.3 Token 有效期太长

错误：

```text
Access Token 30 天
```

泄露后风险极高。

推荐：

```text
Access Token：15 ~ 60 分钟
Refresh Token：7 ~ 30 天
```

---

## 13.4 JWT 里放敏感信息

错误：

```json
{
  "phone": "138xxxx",
  "idCard": "420xxx",
  "password": "xxx"
}
```

JWT Payload 不是加密的。

---

## 13.5 没有校验 issuer / audience

只校验签名还不够。

推荐校验：

```text
iss = devwiki-auth
aud = devwiki-api
```

防止其他系统签发的 Token 被误用。

---

## 13.6 alg 混淆攻击

不要信任 JWT Header 里的 alg。

例如攻击者构造：

```json
{
  "alg": "none"
}
```

或者算法混淆。

正确做法：

```java
Algorithm algorithm = Algorithm.HMAC256(secret);

JWT.require(algorithm)
        .withIssuer("devwiki-auth")
        .build();
```

也就是服务端固定算法，不从客户端 Token 动态选择算法。

---

# 14. 推荐的整体认证架构

```mermaid
flowchart TD
    A[Client] --> B[Auth Service]
    B --> C[(User DB)]
    B --> D[(Redis Refresh Token Store)]
    B --> A

    A --> E[API Gateway]
    E --> F[JWT Verify Filter]
    F --> G[Inject User Headers]
    G --> H[Order Service]
    G --> I[Wiki Service]
    G --> J[AI Service]

    H --> K[(Business DB)]
    I --> L[(Business DB)]
```

核心职责：

|组件|职责|
|---|---|
|Auth Service|登录、签发 Token、刷新 Token、退出登录|
|Gateway|校验 Access Token、注入身份 Header|
|Redis|存 Refresh Token、黑名单、用户状态缓存|
|Business Service|做业务授权|
|DB|存用户、角色、权限、资源归属|

---

# 15. 推荐落地方案

## 第一阶段：MVP

适合个人项目 / 学习项目：

```text
HS256
Access Token 2 小时
Refresh Token 7 天
Gateway 校验 JWT
下游服务读取 X-User-Id
```

优点：简单，能跑通完整链路。

---

## 第二阶段：标准微服务项目

```text
RS256
Access Token 30 分钟
Refresh Token 14 天
Refresh Token 存 Redis
Gateway 验签
Gateway 注入 X-User-Id / X-Tenant-Id / X-Roles
下游服务做业务授权
```

这是比较推荐的企业级结构。

---

## 第三阶段：更高安全要求

```text
RS256 + JWKS
Access Token 15 分钟
Refresh Token Rotation
Token Blacklist
Token Version
设备管理
登录地点审计
风险控制
mTLS
网关到服务内网隔离
```

适合后台管理系统、SaaS、多租户平台。

---

# 16. 面试加分回答

可以这样说：

> JWT 在微服务架构中适合做无状态认证。我的设计通常是 Auth Service 负责登录和签发 Token，API Gateway 负责校验 Access Token 的签名、过期时间、issuer、audience 和 token type，然后把 userId、tenantId、roles 等身份上下文注入到内部 Header。下游服务不再重复解析 JWT，而是基于可信 Header 做业务授权。
> 
> 但是 JWT 不是万能的。它的问题是签发后难以主动失效，所以生产上不会只依赖一个长期 JWT，而是使用短期 Access Token 加长期 Refresh Token。Refresh Token 存 Redis 或数据库，支持退出登录、设备管理和主动吊销。对于更高安全要求，可以结合 jti 黑名单、token version、RS256、JWKS 和密钥轮换。
> 
> 另外，JWT Payload 不是加密的，不能放敏感信息；网关必须先清理外部传入的 X-User-Id 等内部 Header，再重新注入，防止 Header 伪造。

---

# 17. 一句话总结

**JWT 在微服务网关里的最佳实践是：Auth Service 签发短期 Access Token，Gateway 统一验签和注入身份上下文，下游服务基于身份上下文做业务授权；不要把 JWT 当成万能权限系统，也不要让网关承担复杂业务授权。**