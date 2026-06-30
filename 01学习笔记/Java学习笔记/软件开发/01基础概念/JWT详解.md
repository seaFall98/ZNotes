# 1. 一句话解释 JWT

**JWT（JSON Web Token）是一种把用户身份、权限、过期时间等“声明”打包成字符串，并通过签名保证其未被篡改的 Token 格式。**

它常用于：

- 登录后的访问令牌：Access Token
    
- 微服务之间传递用户身份
    
- OAuth2 / OIDC 中的身份令牌：ID Token
    
- 临时授权凭证
    
- 无状态认证方案
    

注意：

> JWT 本身不是登录系统，也不是权限系统。  
> 它只是一个“可验证、可携带、结构化”的令牌格式。

![[ChatGPT Image 2026年6月30日 15_03_28.png]]

---

# 2. JWT 长什么样？

一个典型 JWT 是这样的：

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMDAxIiwidXNlcm5hbWUiOiJ6eiIsInJvbGVzIjpbIkFETUlOIl0sImV4cCI6MTcxOTk5OTk5OX0
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

它由三段组成：

```text
Header.Payload.Signature
```

三段之间用 `.` 分隔。

---

# 3. JWT 的三部分结构

## 3.1 Header：说明令牌类型和签名算法

Header 通常是一个 JSON：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

含义：

|字段|含义|
|---|---|
|`alg`|签名算法，例如 HS256、RS256|
|`typ`|Token 类型，通常是 JWT|

然后 Header 会被 Base64URL 编码成第一段。

---

## 3.2 Payload：真正携带的数据

Payload 也是 JSON，例如：

```json
{
  "sub": "1001",
  "username": "zz",
  "roles": ["ADMIN"],
  "iat": 1719990000,
  "exp": 1719993600
}
```

常见字段：

|字段|全称|含义|
|---|---|---|
|`sub`|Subject|主体，通常是用户 ID|
|`iss`|Issuer|签发者|
|`aud`|Audience|接收方|
|`iat`|Issued At|签发时间|
|`exp`|Expiration Time|过期时间|
|`nbf`|Not Before|在此时间前不可用|
|`jti`|JWT ID|Token 唯一 ID，可用于黑名单|

也可以放自定义字段：

```json
{
  "userId": 1001,
  "tenantId": 2001,
  "roles": ["ADMIN", "USER"],
  "permissions": ["article:create", "article:delete"]
}
```

但要注意：

> Payload 只是 Base64URL 编码，不是加密。  
> 任何拿到 JWT 的人都可以解码看到 Payload 内容。

所以不要放：

- 密码
    
- 身份证号
    
- 手机号
    
- 邮箱等敏感信息
    
- 银行卡信息
    
- 内部密钥
    
- 太多权限细节
    

---

## 3.3 Signature：签名，防止篡改

Signature 的生成逻辑大致是：

```text
签名 = 签名算法(
    base64UrlEncode(Header) + "." + base64UrlEncode(Payload),
    secret 或 privateKey
)
```

以 HS256 为例：

```text
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

最终 JWT：

```text
base64Url(header).base64Url(payload).signature
```

重点：

> JWT 的签名不是为了隐藏内容，而是为了证明内容没有被篡改。

---

# 4. JWT 的核心原理

假设服务端签发了这个 Token：

```json
{
  "sub": "1001",
  "role": "USER",
  "exp": 1719993600
}
```

客户端拿到后，可能想篡改成：

```json
{
  "sub": "1001",
  "role": "ADMIN",
  "exp": 1719993600
}
```

但是只要 Payload 被改了，原来的 Signature 就对不上。

服务端验证时会重新计算签名：

```text
重新计算签名(Header + Payload)
```

如果重新计算出来的签名和 JWT 第三段不一致，说明 Token 被篡改，直接拒绝。

所以 JWT 的安全性来自：

```text
Header + Payload + 密钥/私钥 → Signature
```

而不是来自 Base64 编码。

---

# 5. Base64URL 不是加密

很多人误以为 JWT 被“加密”了，因为它看起来是一长串乱码。

实际上 Header 和 Payload 都可以直接解码。

例如 Payload：

```text
eyJzdWIiOiIxMDAxIiwidXNlcm5hbWUiOiJ6eiJ9
```

解码后就是：

```json
{
  "sub": "1001",
  "username": "zz"
}
```

所以：

|能力|JWT 默认是否具备|
|---|---|
|防篡改|是，靠签名|
|防伪造|是，前提是密钥安全|
|防泄露|否|
|加密内容|否|
|自动失效|否|
|权限控制|否，只能携带权限声明|

如果要让内容不可见，需要用 **JWE**，也就是加密版 Token。普通 JWT 更准确地说是 **JWS：带签名的 JSON Token**。

---

# 6. JWT 的认证流程

## 6.1 登录签发 Token

流程如下：

```text
用户输入账号密码
        ↓
服务端校验账号密码
        ↓
校验成功
        ↓
服务端生成 JWT
        ↓
返回给客户端
        ↓
客户端保存 JWT
```

示例响应：

```json
{
  "accessToken": "xxx.yyy.zzz",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

---

## 6.2 请求接口时携带 JWT

客户端之后请求接口时，一般放在请求头：

```http
Authorization: Bearer xxx.yyy.zzz
```

服务端流程：

```text
收到请求
   ↓
读取 Authorization Header
   ↓
解析 Bearer Token
   ↓
验证签名
   ↓
检查 exp 是否过期
   ↓
检查 iss/aud/权限等声明
   ↓
通过后放行请求
```

---

# 7. JWT 为什么适合“无状态认证”？

传统 Session 登录是这样的：

```text
客户端：Cookie 里保存 sessionId
服务端：Redis / 内存里保存 sessionId 对应的用户信息
```

每次请求：

```text
客户端带 sessionId
服务端查 Redis / Session Store
找到用户信息
```

JWT 登录是这样的：

```text
客户端保存 JWT
JWT 里已经包含用户 ID、过期时间、角色等信息
服务端只需要验证签名即可
```

也就是说：

```text
Session 模式：服务端保存登录状态
JWT 模式：Token 自带状态，服务端只验证
```

---

# 8. JWT vs Session

|对比项|Session|JWT|
|---|---|---|
|状态存储|服务端存储|客户端持有|
|服务端压力|需要查 Session/Redis|可只验证签名|
|横向扩展|需要共享 Session|更容易扩展|
|主动注销|容易，删 Session|麻烦，需要黑名单或版本号|
|Token 泄露风险|sessionId 泄露也危险|JWT 泄露也危险|
|信息携带|sessionId 本身不携带信息|Payload 可携带声明|
|服务端控制力|强|弱|
|适合场景|传统 Web、强管控系统|前后端分离、API、微服务|

JWT 的优势是：

```text
无状态、易扩展、适合分布式系统
```

JWT 的代价是：

```text
一旦签发，在过期前默认很难主动失效
```

---

# 9. JWT 的签名算法

常见算法分两类。

## 9.1 对称签名：HS256

HS256 使用同一个密钥签发和验证。

```text
签发 JWT：使用 secret
验证 JWT：也使用 secret
```

示意：

```text
Auth Server --secret--> 生成 JWT
Resource Server --secret--> 验证 JWT
```

优点：

- 简单
    
- 性能好
    
- 实现容易
    

缺点：

- 所有验证方都必须知道同一个 secret
    
- secret 泄露后，任何服务都能伪造 Token
    
- 不适合开放式多系统验证
    

适合：

- 单体应用
    
- 内部系统
    
- 小型项目
    
- 服务边界清晰的应用
    

---

## 9.2 非对称签名：RS256 / ES256

RS256 使用私钥签发，公钥验证。

```text
签发 JWT：使用 private key
验证 JWT：使用 public key
```

示意：

```text
Auth Server --private key--> 生成 JWT
Resource Server --public key--> 验证 JWT
```

优点：

- 验证方只需要公钥
    
- 公钥泄露也不能伪造 Token
    
- 适合多个服务共同验证
    
- 适合 OAuth2 / OIDC / 微服务体系
    

缺点：

- 实现稍复杂
    
- 性能略低于 HS256
    
- 需要管理密钥轮换
    

适合：

- 微服务系统
    
- 网关统一认证
    
- 第三方系统接入
    
- OAuth2 / OpenID Connect
    
- 企业级认证中心
    

---

# 10. JWT 的典型字段设计

一个较合理的 Access Token Payload：

```json
{
  "iss": "https://auth.example.com",
  "sub": "1001",
  "aud": "zblog-api",
  "iat": 1719990000,
  "exp": 1719990900,
  "jti": "7f3f1e1d-4b98-4d91-8e28-abc123",
  "scope": "article:read article:create",
  "tenantId": "tenant-001"
}
```

解释：

|字段|建议|
|---|---|
|`iss`|必须校验，防止接受错误签发方|
|`sub`|用户唯一 ID，不建议放 username|
|`aud`|必须校验，防止 Token 被错误系统使用|
|`iat`|记录签发时间|
|`exp`|必须设置，且时间不要太长|
|`jti`|可用于黑名单、审计、追踪|
|`scope`|适合放粗粒度权限|
|`tenantId`|多租户系统常用|

不建议放：

```json
{
  "password": "xxx",
  "phone": "xxx",
  "email": "xxx",
  "idCard": "xxx",
  "allUserPermissions": ["几百个权限..."]
}
```

---

# 11. Access Token 和 Refresh Token

生产系统中，JWT 经常和 Refresh Token 搭配使用。

## 11.1 Access Token

Access Token 用来访问接口。

特点：

- 生命周期短
    
- 每次请求都带
    
- 泄露风险高
    
- 一般放在请求头
    

例如：

```text
有效期：5 分钟、15 分钟、30 分钟
```

---

## 11.2 Refresh Token

Refresh Token 用来换新的 Access Token。

特点：

- 生命周期较长
    
- 不应该频繁发送
    
- 安全级别更高
    
- 通常服务端需要保存或可撤销
    

例如：

```text
有效期：7 天、14 天、30 天
```

---

## 11.3 完整流程

```text
用户登录
  ↓
服务端返回 accessToken + refreshToken
  ↓
客户端用 accessToken 请求接口
  ↓
accessToken 过期
  ↓
客户端用 refreshToken 请求刷新接口
  ↓
服务端校验 refreshToken
  ↓
返回新的 accessToken
```

---

## 11.4 为什么不直接让 JWT 有效期很长？

因为 JWT 一旦泄露，在过期前都可能被使用。

所以更合理的是：

```text
短期 Access Token + 长期 Refresh Token
```

例如：

```text
Access Token：15 分钟
Refresh Token：7 天
```

这样即使 Access Token 泄露，攻击窗口也比较短。

---

# 12. JWT 最大的问题：主动失效困难

JWT 的经典问题是：

> 服务端已经签发了 JWT，只要签名有效且没过期，它就能继续使用。

这导致几个场景比较麻烦：

## 12.1 用户主动退出登录

如果是 Session：

```text
删除 Redis 中的 sessionId 即可
```

如果是 JWT：

```text
Token 仍然在客户端手里
只要没过期，理论上还能用
```

解决方案：

|方案|说明|
|---|---|
|短过期时间|Access Token 设置很短|
|Token 黑名单|退出时把 jti 加入 Redis 黑名单|
|用户 tokenVersion|用户改密/退出后提升版本号|
|Refresh Token 可撤销|服务端保存 Refresh Token 状态|
|只让 Refresh Token 有状态|Access Token 短期无状态，Refresh Token 服务端可控|

---

## 12.2 用户改密码后旧 Token 仍可用

解决方法：

在用户表中维护：

```text
passwordChangedAt
tokenVersion
```

验证 JWT 时检查：

```text
JWT.iat 是否早于 passwordChangedAt
JWT.tokenVersion 是否等于用户当前 tokenVersion
```

如果不一致，拒绝。

---

## 12.3 管理员封禁用户后 Token 仍可用

如果系统要求封禁立即生效，就不能完全依赖无状态 JWT。

需要在验证阶段额外查询：

- 用户状态
    
- Token 黑名单
    
- 用户权限版本
    
- 租户状态
    

所以生产系统经常是：

```text
JWT 验签 + 少量服务端状态校验
```

而不是极端的完全无状态。

---

# 13. JWT 存在哪里？

常见存储方式有三种。

## 13.1 localStorage

```js
localStorage.setItem("accessToken", token);
```

优点：

- 简单
    
- 前端容易处理
    
- 页面刷新后仍存在
    

缺点：

- 容易受到 XSS 攻击影响
    
- 一旦页面脚本被注入，Token 可能被窃取
    

适合：

- 安全要求不高的后台系统
    
- 内部工具
    
- 简单前后端分离项目
    

---

## 13.2 sessionStorage

```js
sessionStorage.setItem("accessToken", token);
```

优点：

- 关闭标签页后清除
    
- 比 localStorage 生命周期短
    

缺点：

- 仍然可能受到 XSS 影响
    
- 多标签页共享不方便
    

---

## 13.3 HttpOnly Cookie

服务端设置：

```http
Set-Cookie: access_token=xxx; HttpOnly; Secure; SameSite=Lax
```

优点：

- JavaScript 不能直接读取
    
- 对 XSS 窃取 Token 有较好防护
    

缺点：

- 需要处理 CSRF
    
- 前后端跨域配置更复杂
    
- 移动端/多端 API 处理不如 Header 灵活
    

适合：

- Web 应用
    
- 安全要求较高的浏览器登录态
    
- SSR / BFF 架构
    

---

# 14. Authorization Header vs Cookie

## 14.1 Authorization Header

```http
Authorization: Bearer <access_token>
```

适合：

- 前后端分离 API
    
- App
    
- 小程序
    
- 微服务调用
    
- 开放 API
    

优点：

- 显式传递
    
- 不容易受 CSRF 影响
    
- API 风格清晰
    

缺点：

- 前端通常要能读取 Token
    
- 如果存在 XSS，Token 可能泄露
    

---

## 14.2 Cookie

```http
Cookie: access_token=xxx
```

适合：

- 浏览器 Web 应用
    
- SSR
    
- BFF
    
- 传统网站
    

优点：

- HttpOnly 可防止 JS 读取
    
- 浏览器自动携带
    

缺点：

- 需要防 CSRF
    
- 跨域配置麻烦
    
- 对纯 API 场景不如 Header 直观
    

---

# 15. JWT 的常见安全坑

## 15.1 把敏感信息放进 Payload

错误示例：

```json
{
  "userId": 1001,
  "phone": "138xxxx",
  "password": "123456"
}
```

问题：

```text
Payload 可被任何人解码查看
```

正确做法：

```json
{
  "sub": "1001",
  "scope": "article:read article:create",
  "exp": 1719993600
}
```

---

## 15.2 Token 有效期太长

错误：

```text
Access Token 有效期 30 天
```

问题：

```text
一旦泄露，攻击窗口极长
```

建议：

```text
Access Token：5~30 分钟
Refresh Token：7~30 天，且服务端可撤销
```

---

## 15.3 不校验签名

错误做法：

```java
String payload = decode(jwt);
Long userId = payload.getUserId();
```

这只是解码，不是验证。

正确流程：

```text
解析 Token
验证签名
验证 exp
验证 iss
验证 aud
验证权限
```

---

## 15.4 接受 alg=none

历史上有些库或错误配置会接受：

```json
{
  "alg": "none"
}
```

这意味着没有签名。

正确做法：

```text
服务端必须明确指定允许的算法
不要完全相信 Header 中的 alg
```

---

## 15.5 HS256 / RS256 混淆

如果服务端既支持 HS256，又支持 RS256，且配置不当，可能出现算法混淆风险。

建议：

```text
一个系统明确固定算法
不要根据 JWT Header 动态选择算法
```

例如：

```text
只接受 RS256
拒绝 HS256、none、未知算法
```

---

## 15.6 密钥太短

错误：

```text
secret = "123456"
```

问题：

```text
容易被爆破
```

建议：

```text
使用足够长、随机、高熵的密钥
不同环境使用不同密钥
不要提交到 Git
```

---

## 15.7 不校验 aud / iss

如果不校验 `aud` 和 `iss`，可能导致：

```text
A 系统签发的 Token 被 B 系统错误接受
```

尤其在微服务、多个认证中心、测试环境/生产环境共存时，这是常见问题。

---

## 15.8 前端把 JWT 当作“安全数据库”

错误想法：

```text
我把用户角色、套餐、余额都放 JWT 里，后端就不用查数据库了。
```

问题：

- Token 信息可能过期
    
- 用户权限变更不能实时生效
    
- 封禁、降权、租户停用难以及时反映
    

更合理：

```text
JWT 放身份和粗粒度声明
关键权限和状态仍以服务端为准
```

---

# 16. Java 中如何生成和验证 JWT

下面用 `jjwt` 举例。注意不同版本 API 会有差异，核心思路一样。

## 16.1 添加依赖

```xml
<dependencies>
    <!-- JWT API -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>

    <!-- JWT 实现 -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

    <!-- JSON 序列化支持 -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

## 16.2 生成 JWT

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Date;
import java.util.List;
import java.util.UUID;

public class JwtTokenService {

    // 示例用法：生产环境不要硬编码密钥，应放在环境变量或配置中心
    private final SecretKey secretKey = Keys.hmacShaKeyFor(
            "replace-with-a-very-long-random-secret-key-至少32字节".getBytes(StandardCharsets.UTF_8)
    );

    public String generateAccessToken(Long userId, List<String> roles) {
        Instant now = Instant.now();
        Instant expireAt = now.plusSeconds(15 * 60); // Access Token 有效期 15 分钟

        return Jwts.builder()
                // JWT ID，可用于审计或黑名单
                .id(UUID.randomUUID().toString())

                // subject 一般放用户唯一标识
                .subject(String.valueOf(userId))

                // 签发方
                .issuer("zblog-auth")

                // 接收方
                .audience()
                .add("zblog-api")
                .and()

                // 签发时间和过期时间
                .issuedAt(Date.from(now))
                .expiration(Date.from(expireAt))

                // 自定义声明：建议只放必要的粗粒度信息
                .claim("roles", roles)

                // 使用 HS256 类算法签名
                .signWith(secretKey)

                .compact();
    }
}
```

---

## 16.3 验证 JWT

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;

public class JwtTokenParser {

    private final SecretKey secretKey = Keys.hmacShaKeyFor(
            "replace-with-a-very-long-random-secret-key-至少32字节".getBytes(StandardCharsets.UTF_8)
    );

    public Claims parseAndValidate(String token) {
        return Jwts.parser()
                // 固定验证密钥
                .verifyWith(secretKey)

                // 校验签发方
                .requireIssuer("zblog-auth")

                // 构建 parser
                .build()

                // 解析并验证签名、过期时间等
                .parseSignedClaims(token)

                // 获取 Payload Claims
                .getPayload();
    }
}
```

使用：

```java
Claims claims = jwtTokenParser.parseAndValidate(token);

Long userId = Long.valueOf(claims.getSubject());
Object roles = claims.get("roles");
```

注意：

```text
parseSignedClaims 不是普通 decode，它会验证签名。
```

---

# 17. Spring Security 中 JWT 的位置

在 Spring Security 中，JWT 通常由一个过滤器处理。

请求链路：

```text
HTTP Request
   ↓
JwtAuthenticationFilter
   ↓
读取 Authorization Header
   ↓
验证 JWT
   ↓
构造 Authentication
   ↓
放入 SecurityContext
   ↓
Controller / Service 获取当前用户
```

---

## 17.1 简化版 JWT Filter

```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenParser jwtTokenParser;

    public JwtAuthenticationFilter(JwtTokenParser jwtTokenParser) {
        this.jwtTokenParser = jwtTokenParser;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String authorization = request.getHeader("Authorization");

        // 没有 Token，直接放行，后续由 Spring Security 决定是否拒绝
        if (authorization == null || !authorization.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authorization.substring("Bearer ".length());

        try {
            var claims = jwtTokenParser.parseAndValidate(token);

            Long userId = Long.valueOf(claims.getSubject());

            // 示例：从 claims 中读取角色
            // 生产中建议对类型转换做更严格处理
            List<String> roles = claims.get("roles", List.class);

            var authorities = roles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                    .toList();

            // principal 可以放 userId，也可以放自定义 LoginUser 对象
            var authentication = new UsernamePasswordAuthenticationToken(
                    userId,
                    null,
                    authorities
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);

        } catch (Exception ex) {
            // Token 非法或过期时，清理上下文
            SecurityContextHolder.clearContext();

            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Invalid or expired token");
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

---

# 18. 登录接口大概怎么写？

一个简化版登录流程：

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final UserService userService;
    private final JwtTokenService jwtTokenService;

    public AuthController(UserService userService, JwtTokenService jwtTokenService) {
        this.userService = userService;
        this.jwtTokenService = jwtTokenService;
    }

    @PostMapping("/login")
    public LoginResponse login(@RequestBody LoginRequest request) {
        // 1. 校验用户名密码
        User user = userService.verifyPassword(
                request.username(),
                request.password()
        );

        // 2. 查询用户角色
        List<String> roles = userService.getRoles(user.getId());

        // 3. 生成 Access Token
        String accessToken = jwtTokenService.generateAccessToken(user.getId(), roles);

        // 4. 返回给前端
        return new LoginResponse(
                accessToken,
                "Bearer",
                15 * 60
        );
    }
}
```

DTO：

```java
public record LoginRequest(
        String username,
        String password
) {
}

public record LoginResponse(
        String accessToken,
        String tokenType,
        long expiresIn
) {
}
```

---

# 19. 前端怎么携带 JWT？

## 19.1 请求头携带

```js
const token = localStorage.getItem("accessToken");

fetch("/api/articles", {
  method: "GET",
  headers: {
    "Authorization": `Bearer ${token}`
  }
});
```

---

## 19.2 Axios 拦截器

```js
import axios from "axios";

const api = axios.create({
  baseURL: "/api"
});

// 请求前自动加 Authorization Header
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("accessToken");

  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});

export default api;
```

---

# 20. JWT 和 RBAC 权限控制

JWT 里可以放角色：

```json
{
  "sub": "1001",
  "roles": ["ADMIN"]
}
```

然后后端判断：

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/articles/{id}")
public void deleteArticle(@PathVariable Long id) {
    articleService.delete(id);
}
```

但要注意：

```text
JWT 中的 roles 是签发时的快照。
```

如果管理员后来把用户从 ADMIN 改成 USER，旧 Token 可能仍然带着 ADMIN，直到过期。

所以权限变更要求高的系统，应该：

- 缩短 Access Token 有效期
    
- 使用权限版本号
    
- 关键接口实时查库
    
- 管理后台操作做二次校验
    

---

# 21. JWT 和微服务架构

在微服务里，常见模式是：

```text
客户端
  ↓
API Gateway
  ↓
认证中心签发 JWT
  ↓
各业务服务验证 JWT
```

有两种方式。

---

## 21.1 网关统一验 JWT

```text
Client
  ↓
Gateway 验证 JWT
  ↓
转发 userId / roles 到后端服务
  ↓
Business Service 信任网关
```

优点：

- 业务服务简单
    
- 认证逻辑集中
    
- 网关统一处理鉴权失败
    

缺点：

- 业务服务过度信任网关
    
- 内网调用需要额外保护
    
- 如果绕过网关访问服务会有风险
    

需要配合：

- 内网隔离
    
- 服务间认证
    
- mTLS
    
- 防止外部直连业务服务
    

---

## 21.2 每个服务都验证 JWT

```text
Client
  ↓
Gateway
  ↓
Service A 验 JWT
  ↓
Service B 验 JWT
```

优点：

- 每个服务独立可信
    
- 安全边界更清晰
    
- 不完全依赖网关
    

缺点：

- 各服务都要接入验证逻辑
    
- 公钥/配置分发更复杂
    

企业里常见的是混合方案：

```text
网关做统一认证 + 服务内部做必要的鉴权/身份校验
```

---

# 22. JWT 黑名单怎么做？

如果要支持退出登录立即失效，可以使用 Redis 黑名单。

签发时带 `jti`：

```json
{
  "sub": "1001",
  "jti": "token-uuid",
  "exp": 1719993600
}
```

退出登录时：

```text
将 jti 写入 Redis 黑名单
TTL 设置为 Token 剩余有效期
```

例如：

```text
key: jwt:blacklist:token-uuid
value: 1
ttl: 900 秒
```

验证时：

```text
1. 验证签名
2. 验证 exp
3. 取出 jti
4. 查询 Redis 黑名单
5. 如果存在，拒绝请求
```

示意代码：

```java
public void validateNotBlacklisted(String jti) {
    Boolean exists = redisTemplate.hasKey("jwt:blacklist:" + jti);

    if (Boolean.TRUE.equals(exists)) {
        throw new UnauthorizedException("Token has been revoked");
    }
}
```

代价：

```text
每次请求都要查 Redis，JWT 的完全无状态优势会下降。
```

所以一般只对高安全场景启用，或者只让 Refresh Token 有状态。

---

# 23. 更推荐的生产方案

对于普通前后端分离系统，可以使用：

```text
短期 Access Token + 长期 Refresh Token
```

推荐结构：

```text
Access Token:
- JWT 格式
- 有效期 5~30 分钟
- 不入库
- 每次请求携带

Refresh Token:
- 随机字符串或 JWT
- 有效期 7~30 天
- 服务端保存 hash
- 支持撤销、轮换、设备管理
```

流程：

```text
登录
  ↓
返回 accessToken + refreshToken
  ↓
访问接口使用 accessToken
  ↓
accessToken 过期
  ↓
使用 refreshToken 换新 accessToken
  ↓
refreshToken 轮换
```

Refresh Token 轮换的意思是：

```text
每次刷新都返回新的 refreshToken
旧 refreshToken 立即失效
```

这样可以降低 Refresh Token 被盗用后的风险。

---

# 24. JWT 不适合什么场景？

JWT 不一定适合所有登录系统。

## 不适合场景

|场景|原因|
|---|---|
|强登录态控制|JWT 主动失效较麻烦|
|银行、支付、强风控系统|需要实时风控和服务端状态|
|权限频繁变化系统|Token 权限快照容易滞后|
|Payload 很大的系统|每次请求都携带，浪费带宽|
|需要服务端集中管理会话|Session 更自然|

---

# 25. JWT 适合什么场景？

|场景|原因|
|---|---|
|前后端分离|Header 携带方便|
|移动端 API|不依赖浏览器 Cookie|
|微服务系统|各服务可验证 Token|
|第三方开放接口|可携带签发方、受众、作用域|
|OAuth2 / OIDC|标准体系常用|
|短期访问凭证|签发、验证简单|

---

# 26. JWT 的工程设计建议

## 26.1 Access Token 尽量短

建议：

```text
5~30 分钟
```

不要：

```text
Access Token 有效期 7 天、30 天
```

---

## 26.2 Payload 保持精简

建议放：

```json
{
  "sub": "1001",
  "iss": "zblog-auth",
  "aud": "zblog-api",
  "scope": "article:read article:create",
  "exp": 1719993600
}
```

不要放：

```json
{
  "username": "zz",
  "phone": "xxx",
  "email": "xxx",
  "avatar": "xxx",
  "permissions": ["几百个权限"]
}
```

---

## 26.3 后端必须校验这些字段

至少校验：

```text
签名
exp
iss
aud
算法
```

可选校验：

```text
nbf
jti
tokenVersion
tenantId
clientId
scope
```

---

## 26.4 密钥不要硬编码

不要这样：

```java
private static final String SECRET = "123456";
```

应该：

```text
环境变量
配置中心
KMS
Secret Manager
Kubernetes Secret
```

---

## 26.5 区分 Access Token 和 Refresh Token

不要只有一个长期 JWT。

更合理：

```text
Access Token 短期
Refresh Token 长期且可撤销
```

---

## 26.6 高风险操作重新校验

例如：

- 修改密码
    
- 修改邮箱
    
- 删除账号
    
- 付款
    
- 提现
    
- 删除重要数据
    

不要只看 JWT。

可以要求：

- 重新输入密码
    
- MFA
    
- 短期操作 Token
    
- 后端实时查用户状态
    

---

# 27. JWT 在 Spring Boot 项目中的推荐架构

可以拆成这些模块：

```text
auth
├── AuthController          登录、刷新、退出
├── JwtTokenService         签发 Token
├── JwtTokenParser          验证 Token
├── RefreshTokenService     管理 Refresh Token
├── JwtAuthenticationFilter 请求认证过滤器
├── CurrentUser             当前登录用户对象
└── SecurityConfig          Spring Security 配置
```

核心职责：

|模块|职责|
|---|---|
|`AuthController`|登录、刷新、退出|
|`JwtTokenService`|生成 Access Token|
|`JwtTokenParser`|解析和验证 JWT|
|`RefreshTokenService`|Refresh Token 入库、轮换、撤销|
|`JwtAuthenticationFilter`|从请求中读取 JWT 并放入上下文|
|`SecurityConfig`|配置放行路径、认证规则|
|`CurrentUser`|封装当前用户 ID、角色、租户信息|

---

# 28. 一个完整请求链路

```text
1. 用户登录
   POST /api/auth/login

2. 服务端校验账号密码

3. 服务端签发：
   - accessToken
   - refreshToken

4. 前端保存 Token

5. 请求业务接口：
   GET /api/articles
   Authorization: Bearer accessToken

6. JwtAuthenticationFilter 拦截请求

7. 验证：
   - 签名是否正确
   - Token 是否过期
   - iss 是否正确
   - aud 是否正确
   - 是否在黑名单
   - 用户是否被禁用，视业务要求

8. 构造 Authentication

9. Controller 继续执行

10. Access Token 过期后：
    POST /api/auth/refresh

11. 服务端校验 Refresh Token

12. 返回新的 Access Token
```

---

# 29. JWT 和 OAuth2 / OIDC 的关系

JWT 经常出现在 OAuth2 / OIDC 中，但它们不是一回事。

|概念|含义|
|---|---|
|JWT|Token 格式|
|OAuth2|授权协议|
|OIDC|身份认证协议，建立在 OAuth2 之上|
|Access Token|访问资源的令牌，可能是 JWT，也可能不是|
|ID Token|OIDC 中表示用户身份的令牌，通常是 JWT|
|Refresh Token|用于刷新 Access Token 的令牌|

也就是说：

```text
OAuth2 可以使用 JWT 作为 Access Token 格式
OIDC 的 ID Token 通常就是 JWT
```

但：

```text
不是所有 OAuth2 Access Token 都必须是 JWT
```

---

# 30. 最后总结

JWT 的核心可以压缩成这几句话：

```text
JWT = Header + Payload + Signature
```

```text
Header 说明算法，Payload 携带声明，Signature 保证未被篡改。
```

```text
JWT 默认只是签名，不是加密，所以 Payload 不要放敏感信息。
```

```text
JWT 适合无状态认证和分布式验证，但主动失效、权限实时变更、Token 泄露处理比较麻烦。
```

```text
生产系统不要只用一个长期 JWT，通常应使用短期 Access Token + 可撤销 Refresh Token。
```

```text
服务端验证 JWT 时，不只是 decode，而是必须校验签名、过期时间、签发方、受众、算法和必要的业务状态。
```

最实用的工程结论：

> JWT 适合作为“短期访问凭证”，不适合被当成“永久登录态数据库”。  
> 真正可靠的认证系统通常是：**JWT + Refresh Token + 服务端状态 + 权限校验 + 安全策略**。