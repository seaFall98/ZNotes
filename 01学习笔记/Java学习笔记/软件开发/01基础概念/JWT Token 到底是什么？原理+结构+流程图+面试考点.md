---
title: "全网最透彻：JWT Token 到底是什么？原理+结构+流程图+面试考点"
source: "https://blog.csdn.net/qq_41840843/article/details/159865717"
author:
  - "[[qq_41840843]]"
published: 2026-04-06
created: 2026-06-30
description: "文章浏览阅读4.4k次，点赞5次，收藏9次。在分布式、微服务、前后端分离大行其道的今天，已经成为身份认证、接口鉴权的主流方案。JWT 是什么？JWT 由哪几部分组成？JWT 为什么能实现无状态认证？JWT 安全吗？本文从原理、结构、流程、优缺点、安全实践全方位讲解，让你彻底吃透 JWT。一种开放标准（RFC 7519），用于在网络应用中以 JSON 对象的形式安全传递信息。JWT 是一个自带信息、带签名的无状态身份令牌，适合分布式、微服务、前后端分离项目。登录生成 → 前端存储 → 请求携带 → 验签认证。_jwt token是什么"
tags:
  - "clippings"
---
*🌺The Begin🌺点点关注，收藏不迷路🌺*

### 一、前言

在 分布式 、微服务、前后端分离大行其道的今天， **JWT（JSON Web Token）** 已经成为 **身份认证、接口鉴权** 的主流方案。

面试必问：

- JWT 是什么？
- JWT 由哪几部分组成？
- JWT 为什么能实现无状态认证？
- JWT 安全吗？

本文从 **原理、结构、流程、优缺点、安全实践** 全方位讲解，让你彻底吃透 JWT。

---

### 二、什么是 JWT？

#### 官方定义

**JWT = JSON Web Token**  
一种 **开放标准（RFC 7519）** ，用于在网络应用中 **以 JSON 对象的形式安全传递信息** 。

#### 核心特点

1. **轻量级** ：数据量小，传输快
2. **自包含** ：Token 本身携带用户信息， **服务器不需要存储 Session**
3. **无状态** ：服务端不需要保存任何会话信息
4. **跨域/跨服务** ：适合微服务、分布式、前后端分离

一句话总结：  
**JWT 就是一个加密的、自带用户信息的“身份令牌”，服务端不用存 Session，直接验签即可认证。**

---

### 三、JWT 长什么样？

JWT 是一段由 **3 段 Base64 编码字符串** 组成的文本，用 **.** 分隔：

```
xxxxx.yyyyy.zzzzz
1
```

示例：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
123
```

**三段分别对应：Header.Payload.Signature**

---

### 四、JWT 三大部分结构详解

#### 4.1 Header（头部）

作用：描述 JWT 的 **加密算法、类型**  
示例：

```json
{
  "alg": "HS256",   // 签名算法：HMAC-SHA256
  "typ": "JWT"      // 类型固定为 JWT
}
json1234
```

#### 4.2 Payload（负载）—— 存放数据

作用：存放 **用户信息** （如 userId、username、过期时间）  
**注意：默认是 Base64 编码，不是加密！不要存密码！**

标准字段（推荐使用）：

- **iss** ：签发人
- **exp** ：过期时间
- **sub** ：主题
- **userId** ：用户ID

示例：

```json
{
  "userId": 10086,
  "username": "zhangsan",
  "exp": 1735689600
}
json12345
```

#### 4.3 Signature（签名）—— 最重要

作用： **防止 Token 被篡改**  
生成规则：

```
Signature = HMACSHA256(
  Base64(Header) + "." + Base64(Payload),
  密钥（secret）
)
1234
```

**只有服务器持有密钥，只要签名正确，说明 Token 未被篡改。**

---

### 五、JWT 认证完整流程图（最清晰）

<svg id="mermaid-svg-sCNUY7Mps41LnrZG" width="100%" xmlns="http://www.w3.org/2000/svg" style="max-width: 444.828125px;" viewBox="0.00000762939453125 -0.0000019073486328125 444.828125 821.9765625"><g><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd" viewBox="0 0 10 10" refX="5" refY="5" markerUnits="userSpaceOnUse" markerWidth="8" markerHeight="8" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" style="stroke-width: 1; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointStart" viewBox="0 0 10 10" refX="4.5" refY="5" markerUnits="userSpaceOnUse" markerWidth="8" markerHeight="8" orient="auto"><path d="M 0 5 L 10 10 L 10 0 z" style="stroke-width: 1; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-circleEnd" viewBox="0 0 10 10" refX="11" refY="5" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><circle cx="5" cy="5" r="5" style="stroke-width: 1; stroke-dasharray: 1, 0;"></circle></marker><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-circleStart" viewBox="0 0 10 10" refX="-1" refY="5" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><circle cx="5" cy="5" r="5" style="stroke-width: 1; stroke-dasharray: 1, 0;"></circle></marker><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-crossEnd" viewBox="0 0 11 11" refX="12" refY="5.2" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><path d="M 1,1 l 9,9 M 10,1 l -9,9" style="stroke-width: 2; stroke-dasharray: 1, 0;"></path></marker><marker id="mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-crossStart" viewBox="0 0 11 11" refX="-1" refY="5.2" markerUnits="userSpaceOnUse" markerWidth="11" markerHeight="11" orient="auto"><path d="M 1,1 l 9,9 M 10,1 l -9,9" style="stroke-width: 2; stroke-dasharray: 1, 0;"></path></marker><g><g></g><g><path d="M229.712,61.997L229.712,66.164C229.712,70.331,229.712,78.664,229.712,86.331C229.712,93.997,229.712,100.997,229.712,104.497L229.712,107.997" id="L_A_B_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M229.712,165.995L229.712,170.161C229.712,174.328,229.712,182.661,229.712,190.328C229.712,197.995,229.712,204.995,229.712,208.495L229.712,211.995" id="L_B_C_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M229.712,269.992L229.712,274.159C229.712,278.326,229.712,286.659,229.712,294.326C229.712,301.992,229.712,308.992,229.712,312.492L229.712,315.992" id="L_C_D_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M229.712,373.99L229.712,378.156C229.712,382.323,229.712,390.656,229.712,398.323C229.712,405.99,229.712,412.99,229.712,416.49L229.712,419.99" id="L_D_E_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M229.712,477.987L229.712,482.154C229.712,486.32,229.712,494.654,229.712,502.32C229.712,509.987,229.712,516.987,229.712,520.487L229.712,523.987" id="L_E_F_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M229.712,581.984L229.712,586.151C229.712,590.318,229.712,598.651,229.712,606.318C229.712,613.984,229.712,620.984,229.712,624.484L229.712,627.984" id="L_F_G_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M179.211,685.982L167.677,692.148C156.143,698.315,133.074,710.648,121.54,722.314C110.005,733.98,110.005,744.98,110.005,750.479L110.005,755.979" id="L_G_H_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path><path d="M280.213,685.982L291.748,692.148C303.282,698.315,326.351,710.648,337.885,722.314C349.419,733.98,349.419,744.98,349.419,750.479L349.419,755.979" id="L_G_I_0" style=";" marker-end="url(#mermaid-svg-sCNUY7Mps41LnrZG_flowchart-v2-pointEnd)" fill="none" stroke="currentColor"></path></g><g><g><g transform="translate(0, 0)"></g></g><g><g transform="translate(0, 0)"></g></g><g><g transform="translate(0, 0)"></g></g><g><g transform="translate(0, 0)"></g></g><g><g transform="translate(0, 0)"></g></g><g><g transform="translate(0, 0)"></g></g><g transform="translate(110.00520324707031, 722.9804592132568)"><g transform="translate(-16.00260353088379, -11.998697280883789)"><foreignObject width="32.00520706176758" height="23.997394561767578"><p>合法</p></foreignObject></g></g><g transform="translate(349.4192581176758, 722.9804592132568)"><g transform="translate(-36.197914123535156, -11.998697280883789)"><foreignObject width="72.39582824707031" height="23.997394561767578"><p>非法/过期</p></foreignObject></g></g></g><g><g id="flowchart-A-0" transform="translate(229.71223068237305, 34.99869728088379)"><rect style="" x="-118.00129699707031" y="-26.99869728088379" width="236.00259399414062" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-88.00129699707031, -11.998697280883789)"><rect></rect><foreignObject width="176.00259399414062" height="23.997394561767578"><p>客户端输入账号密码登录</p></foreignObject></g></g><g id="flowchart-B-1" transform="translate(229.71223068237305, 138.99609184265137)"><rect style="" x="-102.00520324707031" y="-26.99869728088379" width="204.01040649414062" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-72.00520324707031, -11.998697280883789)"><rect></rect><foreignObject width="144.01040649414062" height="23.997394561767578"><p>服务器验证账号密码</p></foreignObject></g></g><g id="flowchart-C-3" transform="translate(229.71223068237305, 242.99348640441895)"><rect style="" x="-173.3528594970703" y="-26.99869728088379" width="346.7057189941406" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-143.3528594970703, -11.998697280883789)"><rect></rect></g></g><g id="flowchart-D-5" transform="translate(229.71223068237305, 346.9908809661865)"><rect style="" x="-121.95313262939453" y="-26.99869728088379" width="243.90626525878906" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-91.95313262939453, -11.998697280883789)"><rect></rect><foreignObject width="183.90626525878906" height="23.997394561767578"><p>服务器返回 JWT 给客户端</p></foreignObject></g></g><g id="flowchart-E-7" transform="translate(229.71223068237305, 450.9882755279541)"><rect style="" x="-168.11196899414062" y="-26.99869728088379" width="336.22393798828125" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-138.11196899414062, -11.998697280883789)"><rect></rect></g></g><g id="flowchart-F-9" transform="translate(229.71223068237305, 554.9856700897217)"><rect style="" x="-191.5559844970703" y="-26.99869728088379" width="383.1119689941406" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-161.5559844970703, -11.998697280883789)"><rect></rect><foreignObject width="323.1119689941406" height="23.997394561767578"><p>下次请求：请求头 Authorization: Bearer JWT</p></foreignObject></g></g><g id="flowchart-G-11" transform="translate(229.71223068237305, 658.9830646514893)"><rect style="" x="-137.94921875" y="-26.99869728088379" width="275.8984375" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-107.94921875, -11.998697280883789)"><rect></rect><foreignObject width="215.8984375" height="23.997394561767578"><p>服务器校验 JWT 签名是否正确</p></foreignObject></g></g><g id="flowchart-H-13" transform="translate(110.00520324707031, 786.9778537750244)"><rect style="" x="-102.00520324707031" y="-26.99869728088379" width="204.01040649414062" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-72.00520324707031, -11.998697280883789)"><rect></rect><foreignObject width="144.01040649414062" height="23.997394561767578"><p>通过认证，返回数据</p></foreignObject></g></g><g id="flowchart-I-15" transform="translate(349.4192581176758, 786.9778537750244)"><rect style="" x="-87.40885162353516" y="-26.99869728088379" width="174.8177032470703" height="53.99739456176758" fill="none" stroke="currentColor"></rect><g style="" transform="translate(-57.408851623535156, -11.998697280883789)"><rect></rect><foreignObject width="114.81770324707031" height="23.997394561767578"><p>返回 401 未授权</p></foreignObject></g></g></g></g></g></svg>

---

### 六、JWT 认证步骤（序号版）

1. 用户登录，发送账号密码
2. 服务端验证成功
3. 服务端根据用户信息生成 **JWT Token**
4. 服务端把 JWT 返回给前端
5. 前端存储 JWT（localStorage / Cookie）
6. 前端每次请求在 **请求头** 带上 JWT
7. 服务端 **验证签名**
	- 签名正确 → 合法用户
		- 签名错误 → 伪造/篡改，拒绝
8. 完成鉴权

---

### 七、JWT 为什么比 Session 好？（核心优势）

1. **无状态**  
	服务端不需要存 Session，内存无压力
2. **天然支持分布式/微服务**  
	不需要 Session 共享（Redis）
3. **跨域跨平台极强**  
	支持 APP、小程序、Web、第三方服务
4. **自包含**  
	一次携带信息，不用查库
5. **适合 RESTful API**

---

### 八、JWT 缺点与注意事项（面试必问）

1. **Payload 可解码，不能存敏感信息（密码）**
2. **一旦签发，无法中途作废** （除非用黑名单）
3. **过期时间不能太长** ，否则不安全
4. **密钥 secret 绝对不能泄露**
5. 体积比 SessionId 大，每次请求携带消耗流量

---

### 九、高频面试题（满分答案）

#### 9.1 JWT 由哪三部分组成？

**Header（头部）、Payload（负载）、Signature（签名）**

#### 9.2 JWT 为什么安全？

**因为有签名，只要密钥不泄露，Token 无法被篡改。**

#### 9.3 JWT 存密码可以吗？

**不可以！Payload 是 Base64 编码，可解码，不是加密。**

#### 9.4 JWT 与 Session 区别？

- JWT：无状态、服务端不存储、跨域强
- Session：有状态、服务端存储、分布式麻烦

#### 9.5 JWT 如何作废？

无法直接作废，只能：

---

### 十、总结

#### JWT 核心一句话

**JWT 是一个自带信息、带签名的无状态身份令牌，适合分布式、微服务、前后端分离项目。**

结构： **Header.Payload.Signature**  
流程： **登录生成 → 前端存储 → 请求携带 → 验签认证**

---

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2d66846c0fbf4e1181e67009633c918c.png#pic_center)

*🌺The End🌺点点关注，收藏不迷路🌺*

微信名片