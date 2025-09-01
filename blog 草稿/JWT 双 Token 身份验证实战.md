# Cookies + Session 身份验证
web 项目中传统的身份验证方式是 Cookies + Session。客户端携带身份验证信息请求服务端，服务端校验通过后，将用户信息存储在 Session 中，并将 Session Id 通过 Set-Cookies 方式写入响应头，客户端接收到响应后存储 Cookies，下次请求服务端时，根据 Cookies 规则，浏览器自动携带 Cookies，服务端通过 Cookies 中的 Session Id 获取用户对应的 Session 会话。

>**Cookie** 是由服务端设置，存储在浏览器端的小型文本信息，浏览器在后续请求时会自动携带，用于标识用户或保存客户端状态。  
>**Session** 是服务端保存的用户会话信息，用于弥补 HTTP 无状态的缺陷，实现对用户身份、权限、登录状态等的管理。

传统的身份验证由于依赖浏览器 Cookies，移动端适配性较差，且在分布式环境下单机 Session 无法有效管理用户会话。一种解决思路是共享 Session。利用 Redis 存储 Session 信息，所有服务端共享 Session。

# JWT 身份验证
服务端存储 Session 方案中，服务端存储会话状态，每次都需要根据请求中携带的 Cookies 信息查找对应的 Session 会话，压力较大，且单机 Session 对分布式不友好。

JWT (JSON Web Token) 基于 JSON 串，提供了**无状态、可验证**的令牌，可以保存传递信息，适合身份验证和信息交换。

JWT 包含 Header、Payload 和 Signature 三个部分。Header 中包含了签名算法和 Token 类型；Payload 负载中存储一定量信息；Signature 是利用服务端私钥和签名算法对 Header 和 Payload 串计算得出的签名串。

## JWT 生成与验证
JWT 的生成与验证完全通过计算签名实现，且 JWT 载荷中已经包含了用户相关信息，客户端携带 JWT 请求服务端，服务端仅需从 JWT 中解码所需内容即可视作该请求已经通过身份验证。现在来描述 JWT 生成与 JWT 验证流程。

判断 JWT 是否被篡改的标准完全依赖签名值，而签名完全依赖于签名算法和服务端私钥。一旦私钥暴露，服务端无法鉴别 JWT 是否已经被篡改。
### JWT 生成
可以简单概述为以下流程：
1. 利用 Base64URL 对 Header 和 Payload 进行编码分别得到 Header 串和 PayLoad 串。
2. 利用服务端私钥和 Header 中定义的签名算法对 Header 串和 Payload 串进行计算得到 Signature 签名串。
3. 使用 '.' 组合 Header 串、Payload 串和 Signature 串得到 JWT。

**注意**：JWT 负载内容并未加密，而是被编码为字符串，任意 JWT 串都可以直接解码出负载内容。JWT 安全传输需结合 HTTPS 协议，并且 JWT 中不应该存储敏感信息。

### JWT 验证
服务端使用服务端私钥计算 JWT 签名以验证其是否被篡改，大致流程如下：
1. 解码 Header 串获取签名算法。
2. 截取 Header 串.Payload 串，利用服务端私钥和签名算法计算新 Signature 签名。
3. 比对新签名和旧 Signature 签名串是否一致，一致则说明 JWT 为被篡改，否则被篡改。

### JWT 特点
JWT 是服务端颁发给客户端的令牌，其特性如下：
- **无状态（Stateless）**  
    服务端不需要存储会话数据，所有信息都在 JWT 内，便于横向扩展、微服务架构。
- **自包含（Self-contained）**  
    JWT 本身携带用户标识、权限、过期时间等数据，可减少数据库查询。
- **跨域友好**  
    可以放在 HTTP Header（`Authorization: Bearer <token>`）传输，不依赖 Cookie，适合 API 和移动端。

服务器不会保存 JWT 的状态信息，单纯依赖 JWT 服务有效控制用户登录状态，比如登出和被踢下线等。并且由于 JWT 自包含，不依赖服务端状态，泄露后危害极大，因此必须限制 JWT 的有效期，单纯依赖 JWT 进行身份验证会导致用户频繁登录，影响体验。即单纯依赖 JWT 进行身份验证难以控制用户登录状态，且存在 JWT 频繁过期的问题。

## RefreshToken 续签和 JWT 黑名单
为了控制用户登录状态以及解决 JWT 频繁过期的问题，引入了 JWT 黑名单和 RefreshToken 续签。

RefreshToken 续签流程如下：
1. 客户端携带身份信息请求服务端
2. 服务端校验通过后签发 JWT 并生成 RefreshToken，RefreshToken 中记录了用户 Id 、Jti、有效期等内容。
3. 当 JWT 过期后，客户端携带 RefreshToken 请求服务端，服务端验证 Redis 中的 RefreshToken，验证通过后，删除旧的 RefreshToken 生成新的 RefreshToken 和 JWT 并发送给客户端。
4. 客户端携带新的 JWT 正常请求服务端。

JWT 黑名单：
1. 用户登出后，将 Jti 加入 JWT 黑名单，移除 RefreshToken。
2. 服务端验证 JWT 时增加校验 JWT 是否处于黑名单的逻辑，如果是，JWT 无效，否则有效。

## JWT 存储位置

服务端签发 JWT 后，客户端可以将 JWT 存储在 Cookie 或者浏览器 LocalStorage 或 SessionStorage：
 1. **存储在浏览器 Cookie**
	JWT 可以存储在 Cookie 中，优点是浏览器会自动随请求发送，无需手动添加请求头，并且可以通过设置 `HttpOnly`、`Secure`、`SameSite` 属性来提升安全性，防止 XSS 窃取和部分 CSRF 攻击。然而，它仍可能遭受 CSRF 攻击，需要结合 CSRF Token 或使用严格的 SameSite 模式来防护，同时前端对其控制能力较弱。
 2. **存储在浏览器 LocalStorage / SessionStorage**
	存储在 `localStorage` 或 `sessionStorage` 中，前端可完全控制存取，操作灵活，并且不会自动随请求发送，天然降低了 CSRF 风险。但由于 JavaScript 可直接访问这些存储位置，JWT 容易在发生 XSS 攻击时被窃取，而且每次请求都需要手动将 JWT 放入 `Authorization: Bearer <token>` 请求头中。

>**CSRF Cross-Site Request Forgery**（跨站请求伪造）：攻击者诱导用户在已登录状态下访问恶意网站，利用浏览器自动携带 Cookie 的特性，伪装成用户对目标站点发起请求，从而执行未授权操作。
>**XSS：Cross-Site Scripting**（跨站脚本攻击）：攻击者将恶意脚本注入网页（如评论、搜索框），当其他用户浏览该页面时，脚本会在浏览器执行，从而窃取信息（如 JWT）、篡改页面或冒充用户操作。