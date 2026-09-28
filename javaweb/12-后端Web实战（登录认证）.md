# 12 后端 Web 实战：登录认证

本章把“用户登录后才能访问系统”做成一个完整功能。学习重点不是背代码，而是理解一条完整链路：登录接口如何验证用户、服务器如何识别后续请求、前端如何携带身份信息，以及未登录时如何被拦截。

## 1. 学习目标与前置知识

### 1.1 学完本章应该会什么

- 能解释 Cookie、Session、Token 和 JWT 的区别。
- 能实现登录接口并返回统一响应结果。
- 能使用 JWT 表示用户身份。
- 能使用 Filter 或 Interceptor 统一校验请求。
- 能处理“未登录返回 401”的前后端流程。

### 1.2 前置知识

建议先掌握已有笔记中的 HTTP 请求响应、Spring Boot Controller、三层架构、MyBatis 基础和 Axios。对应《JavaWeb笔记》的“SpringBootWeb 请求响应”“登录认证”章节。

## 2. 登录功能

### 2.1 业务流程

1. 用户在登录页面输入用户名和密码。
2. 前端发送 `POST /login`，请求体通常是 JSON。
3. Controller 接收参数，Service 查询数据库并校验密码。
4. 校验成功后生成身份凭证。
5. 服务端返回统一结果，前端保存凭证并跳转首页。

登录只负责“确认你是谁”；登录校验负责“后续请求是否仍然是这个用户”。

### 2.2 接口示例

```http
POST /login
Content-Type: application/json

{
  "username": "admin",
  "password": "123456"
}
```

成功响应可以统一为：

```json
{
  "code": 1,
  "msg": "success",
  "data": "eyJhbGciOiJIUzI1NiJ9..."
}
```

失败时不要泄露“用户名不存在”还是“密码错误”等过多细节，通常返回统一的登录失败提示。

### 2.3 分层实现思路

- Controller：接收请求、调用 Service、返回结果。
- Service：查询用户、校验密码、生成 JWT。
- Mapper：只负责数据库查询。
- 工具类：负责 JWT 生成和解析。

不要在 Controller 中直接写 SQL，也不要把 JWT 密钥散落在多个类中。

## 3. 会话与会话跟踪

HTTP 默认是无状态的：服务器处理完一次请求后，不会自动记住下一次请求是谁。会话跟踪就是让服务器能够把多次请求关联到同一个用户。

### 3.1 Cookie

Cookie 是浏览器保存并在请求中自动携带的小段数据。优点是使用简单；缺点是容量有限，并且需要注意 `HttpOnly`、`Secure`、`SameSite` 等安全属性。

### 3.2 Session

Session 把用户状态保存在服务器，浏览器只保存 Session ID。优点是服务端可以主动失效会话；缺点是多台服务器需要共享 Session 或做会话粘滞，扩展性较复杂。

### 3.3 Token

Token 是服务端签发给客户端的身份字符串，客户端在后续请求中主动放到请求头。服务器可以不保存每个用户的会话状态，适合前后端分离项目。

| 方案 | 状态保存位置 | 常见使用场景 | 注意点 |
|---|---|---|---|
| Cookie | 浏览器 | 简单网站、配合 Session | 防 CSRF、设置安全属性 |
| Session | 服务端 | 传统服务端渲染 | 集群共享和过期管理 |
| Token | 客户端携带、服务端验证 | 前后端分离、移动端 | 防泄露、设置过期时间 |

## 4. JWT

JWT（JSON Web Token）通常由三段组成：Header、Payload、Signature，使用点号连接。

```text
header.payload.signature
```

- Header：算法和令牌类型。
- Payload：用户 ID、用户名、过期时间等声明。
- Signature：用密钥对前两段签名，防止内容被篡改。

JWT 的 Payload 不是加密内容，不要放密码、身份证号等敏感信息。签名能验证“是否被改过”，不能让内容保密。

### 4.1 生成和校验

```java
String token = JwtUtil.createToken(empId, username);

Claims claims = JwtUtil.parseToken(token);
Long id = Long.valueOf(claims.get("id").toString());
```

实际项目中应把密钥放在配置文件或环境变量中，并设置明确的过期时间。密钥泄露后，攻击者可能伪造令牌。

### 4.2 登录时下发令牌

登录成功后服务端生成 JWT，放入统一响应的 `data` 字段。前端保存到 `localStorage` 或其他合适位置，之后通过 `Authorization` 请求头携带：

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

项目也可能直接使用自定义请求头，例如 `token`。前后端必须提前约定名称和格式。

## 5. Filter 登录校验

Filter 是 Servlet 层的请求过滤器，可以在请求到达 Controller 前统一执行。登录接口、静态资源和健康检查接口通常需要放行。

```java
@WebFilter(urlPatterns = "/*")
public class LoginCheckFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;

        String uri = req.getRequestURI();
        if (uri.contains("/login")) {
            chain.doFilter(request, response);
            return;
        }

        String token = req.getHeader("Authorization");
        try {
            JwtUtil.parseToken(token.replace("Bearer ", ""));
            chain.doFilter(request, response);
        } catch (Exception e) {
            resp.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        }
    }
}
```

示例用于理解流程，生产代码还需要处理空 Token、响应 JSON、跨域和日志等问题。

如果使用 `@WebFilter`，还要确保启动类开启 Servlet 组件扫描：

```java
@SpringBootApplication
@ServletComponentScan
public class TliasApplication { }
```

过滤器返回失败时应设置 `Content-Type: application/json;charset=UTF-8` 并写出统一错误结构，而不是只设置状态码后让浏览器收到空响应。对 `OPTIONS` 预检请求可在 CORS 配置中先处理，再进入业务认证逻辑。

## 6. Interceptor 登录校验

Interceptor 是 Spring MVC 提供的拦截机制，通常实现 `HandlerInterceptor`，并在配置类中注册。相比 Filter，它更接近 Controller，可以获取处理器信息。

```java
@Component
public class LoginCheckInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        String token = request.getHeader("Authorization");
        try {
            JwtUtil.parseToken(token.replace("Bearer ", ""));
            return true;
        } catch (Exception e) {
            response.setStatus(401);
            return false;
        }
    }
}
```

```java
registry.addInterceptor(loginCheckInterceptor)
        .addPathPatterns("/**")
        .excludePathPatterns("/login", "/error");
```

### 6.1 Filter 与 Interceptor 的区别

| 对比项 | Filter | Interceptor |
|---|---|---|
| 所属层次 | Servlet | Spring MVC |
| 拦截范围 | 更底层、更广 | Controller 请求 |
| 常见用途 | 编码、跨域、全局过滤 | 登录校验、权限、操作前后处理 |
| 是否能获取处理器 | 不直接面向 Controller | 可以获取 Handler |

## 7. 前端配合

前端通常用 Axios 请求拦截器自动携带 JWT，用响应拦截器统一处理 401：

```javascript
request.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

request.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      router.push('/login')
    }
    return Promise.reject(error)
  }
)
```

## 8. 小白易错点

- JWT 不是加密，Payload 不能放敏感信息。
- 登录接口必须放行，否则会出现“登录也需要登录”的死循环。
- 前端保存了 Token，不代表服务端一定认可；服务端必须验证签名和过期时间。
- 401 表示未认证，403 通常表示已认证但没有权限。
- Token 解析失败、过期、请求头名称不一致，都会导致校验失败。

## 9. 练习清单

1. 用 Apifox 调通登录接口。
2. 手动删除 Token，观察访问员工列表的响应。
3. 修改 JWT 其中一段，确认服务端拒绝请求。
4. 分别用 Filter 和 Interceptor 实现登录校验，并比较执行时机。
5. 在前端 Axios 中实现 Token 自动携带和 401 跳转。

## 10. 与其他资料的对应关系

- 《JavaWeb笔记》：第十四章“登录认证”。
- 本仓库按 01 到 20 的顺序整理这套课程笔记，本篇是第 12 篇；第 10 篇“日志、认证和请求拦截”可作为本章前置回顾。
- 后续第 13 章会使用本章的 JWT 和 ThreadLocal 记录当前登录员工。

## 11. 关键补充：一次请求到底怎样完成认证

登录认证要拆成两个阶段：第一次请求负责“签发凭证”，后续每一次受保护请求负责“验证凭证”。不要把“登录成功”误认为“以后永久登录”。

```mermaid
flowchart TD
    A[登录页输入用户名密码] --> B[POST /login]
    B --> C{用户是否存在且密码正确}
    C -- 否 --> D[返回 401 或业务失败码]
    C -- 是 --> E[生成 JWT: sub/exp 等声明]
    E --> F[返回统一响应 data=token]
    F --> G[前端保存 token]
    G --> H[请求业务接口]
    H --> I[请求头携带 Authorization]
    I --> J{Filter/Interceptor 验证签名和 exp}
    J -- 失败 --> K[返回 401]
    J -- 成功 --> L[写入 ThreadLocal 并放行]
    L --> M[Controller/Service 执行业务]
    M --> N[afterCompletion 清理 ThreadLocal]
```

### 11.1 密码校验不能照抄明文方案

课程示例为了突出登录流程，数据库中可能直接出现 `123456`。真实项目应保存密码哈希，而不是保存原密码：

```java
// 注册或初始化管理员时
String encoded = passwordEncoder.encode(rawPassword);

// 登录时
if (!passwordEncoder.matches(rawPassword, employee.getPassword())) {
    throw new RuntimeException("用户名或密码错误");
}
```

推荐使用 BCrypt/Argon2 这类带随机盐的密码哈希；密码哈希和 JWT 签名密钥是两回事，前者用于比对密码，后者用于验证令牌完整性。

### 11.2 JWT 校验清单

服务端至少检查以下项目：

1. 请求头是否存在，是否能正确去掉 `Bearer ` 前缀。
2. 签名算法是否是系统允许的算法，签名密钥是否来自安全配置。
3. `exp` 是否过期，`nbf`/`iat` 等时间声明是否合理。
4. `sub` 或自定义 `id` 是否存在、类型是否正确。
5. 用户是否仍处于有效状态（例如被禁用）。

JWT 的三个部分只是 Base64URL 编码，不是加密。RFC 7519 定义的是令牌格式；具体的密钥保管、算法选择和撤销策略仍然由应用负责。

### 11.3 401、403、登录失效的处理边界

| 情况 | 后端状态码 | 前端动作 |
|---|---:|---|
| 没有 Token、Token 过期或签名错误 | 401 | 清除本地 Token，跳转登录页 |
| 已认证但没有操作权限 | 403 | 保留登录状态，提示无权限 |
| 登录接口账号密码错误 | 401 或约定业务码 | 留在登录页并提示 |
| 网络超时、服务器不可达 | 无响应/5xx | 提示网络或服务异常，不要误删 Token |

### 11.4 Filter 与 Interceptor 的执行时序

```mermaid
sequenceDiagram
    participant B as 浏览器
    participant F as Filter
    participant I as Interceptor
    participant C as Controller
    participant S as Service
    B->>F: HTTP 请求
    F->>F: 读取并解析 Token
    alt Token 非法
        F-->>B: 401
    else Token 合法
        F->>I: 放行到 Spring MVC
        I->>I: preHandle
        I->>C: 调用处理方法
        C->>S: 业务调用
        S-->>C: 返回结果
        C-->>I: 返回响应
        I->>I: postHandle/afterCompletion
        I-->>F: 响应返回
        F-->>B: JSON 响应
    end
```

若在 Filter 中写入 `ThreadLocal`，清理动作应放在 `finally` 中；若在 Interceptor 中写入，清理动作应放在 `afterCompletion`，这样异常响应也能执行清理。

### 11.5 跨域和预检请求

前后端端口不同时，浏览器可能先发送 `OPTIONS` 预检请求。认证过滤器不能把合法的预检请求当成普通业务请求拦掉；同时服务端要正确返回 `Access-Control-Allow-Origin`、`Access-Control-Allow-Headers` 等 CORS 响应头。允许携带凭证时不能使用通配符 `*` 作为允许来源。

## 12. 本章总结

1. 登录只解决“确认你是谁”，登录校验解决“后续每次请求是否仍是同一个用户”，两个阶段要分开设计。
2. HTTP 本身无状态，Cookie、Session、Token 都是会话跟踪方案：Cookie 存在浏览器、Session 存在服务端、Token 由客户端携带服务端验证，前后端分离项目通常选 Token。
3. JWT 由 Header、Payload、Signature 三段用点号连接，前两段只是 Base64URL 编码而不是加密，签名只能证明没被改过，Payload 可被任何人解出，不能放密码、身份证号等敏感信息。
4. 登录成功后服务端把 JWT 放进统一响应的 `data` 字段，前端保存到 `localStorage`，之后用 `Authorization: Bearer <token>` 请求头携带，请求头名称和格式必须前后端提前约定。
5. 服务端校验 JWT 至少要看请求头格式、签名算法与密钥来源、`exp` 等时间声明、用户标识声明以及用户是否仍然有效，密钥应放在配置文件或环境变量中并设置明确过期时间。
6. 密码不能明文比对，应保存 BCrypt/Argon2 这类带随机盐的哈希；密码哈希用于比对密码，JWT 签名密钥用于验证令牌完整性，两者不能混为一谈。
7. Filter 属于 Servlet 层，拦截范围更底层更广，Interceptor 属于 Spring MVC，能获取 Handler 信息；登录校验两者都能实现，但执行时机和配置方式不同。
8. 登录接口、静态资源和健康检查接口必须放行，否则会出现“登录也需要登录”的死循环；Filter 校验失败时应返回统一 JSON 错误结构并设置 `Content-Type: application/json;charset=UTF-8`。
9. 前端用 Axios 请求拦截器自动携带 Token，用响应拦截器统一处理 401；401 表示未认证，403 表示已认证但没有权限，网络超时或服务不可达时不应误删本地 Token。
10. 在 Filter 或 Interceptor 中写入 ThreadLocal 后必须在 `finally` 或 `afterCompletion` 中清理，前后端端口不同时还要放行 `OPTIONS` 预检请求并正确返回 CORS 响应头。

## 13. 联网核对与延伸阅读

- [RFC 7519：JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [Spring Boot 外部化配置](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Vue Router 导航守卫](https://router.vuejs.org/guide/advanced/navigation-guards.html)
