# 07 后端 Web 实战（部门管理）

## 1. 项目规范

### 1.1 REST 接口

| 功能 | 方法 | 路径 |
|---|---|---|
| 查询全部 | GET | /depts |
| 查询单个 | GET | /depts/{id} |
| 新增 | POST | /depts |
| 修改 | PUT | /depts |
| 删除 | DELETE | /depts/{id} |

REST 是风格约定：URL 定位资源，HTTP 方法描述操作。

REST 接口设计原则：

- URL 使用名词表示资源，例如 `/depts`、`/emps`，不要把动作写成 `/queryDept`。
- GET 用于查询，POST 用于新增，PUT 用于修改，DELETE 用于删除。
- 同一资源的不同操作通过 HTTP 方法区分。
- 接口返回统一 JSON 结构，前端不需要为每个接口猜测格式。

GET、PUT 和 DELETE 通常具有幂等性：重复执行最终结果相同；POST 通常用于新增，重复提交可能产生多条数据，所以新增接口要结合唯一约束或幂等设计。

接口文档至少要写清请求方法、URL、路径参数、查询参数、请求体、响应体、状态码和错误场景。前后端必须约定字段命名和时间格式，不能只凭接口截图猜测。本章统一使用 `code`、`msg`、`data` 三个字段，全部示例和说明都按这三个名字写，不要在别处再出现 `message` 这种写法。

### 1.2 开发流程

~~~mermaid
flowchart TD
    A[需求分析] --> B[定义接口]
    B --> C[后端实现]
    B --> D[前端实现]
    C --> E[Apifox 测试]
    D --> F[页面测试]
    E --> G[前后端联调]
    F --> G
~~~

## 2. 部门 CRUD

### 2.1 接口约定

| 接口 | 请求参数 | 成功结果 |
|---|---|---|
| `GET /depts` | 无或查询条件 | 部门集合 |
| `GET /depts/{id}` | 路径参数 `id` | 单个部门 |
| `POST /depts` | JSON 请求体 | 新增成功 |
| `PUT /depts` | JSON 请求体，包含 `id` | 修改成功 |
| `DELETE /depts/{id}` | 路径参数 `id` | 删除成功 |

统一响应对象只使用 `code`、`msg` 和 `data` 三个字段，不要一会儿写 `message`、一会儿写 `msg`，字段名不一致会让前端到处做兼容。

| 字段 | 类型 | 含义 | 约定 |
|---|---|---|---|
| `code` | int | 业务结果码 | 成功为 1，业务失败为 0 |
| `msg` | String | 提示信息 | 成功是 `success`，失败是给用户看的原因 |
| `data` | 泛型 | 业务数据 | 查询返回对象或集合，增删改成功时为 `null` |

~~~java
public record Result<T>(int code, String msg, T data) {
    public static <T> Result<T> success(T data) {
        return new Result<>(1, "success", data);
    }

    public static Result<Void> success() {
        return new Result<>(1, "success", null);
    }

    public static Result<Void> error(String msg) {
        return new Result<>(0, msg, null);
    }
}
~~~

前端只需要判断 `code` 是否为 1，失败时直接展示 `msg`。HTTP 状态码仍然表达认证、参数和服务器错误，不能用业务 `code` 代替 HTTP 状态码：资源不存在时既可以返回 404，也可以返回 200 加 `code=0`，两种风格都要在接口文档里写清楚，不能混用。

### 2.2 Mapper

~~~java
@Mapper
public interface DeptMapper {
    @Select("SELECT id, name, create_time, update_time FROM dept ORDER BY id DESC")
    List<Dept> findAll();

    @Insert("INSERT INTO dept(name, create_time, update_time) VALUES(#{name}, NOW(), NOW())")
    void insert(Dept dept);

    @Update("UPDATE dept SET name=#{name}, update_time=NOW() WHERE id=#{id}")
    void update(Dept dept);

    @Delete("DELETE FROM dept WHERE id=#{id}")
    void deleteById(Long id);
}
~~~

### 2.3 Service

Service 负责组织业务逻辑，Controller 只负责接收参数和返回结果：

~~~java
@Service
public class DeptServiceImpl implements DeptService {
    private final DeptMapper mapper;

    public DeptServiceImpl(DeptMapper mapper) {
        this.mapper = mapper;
    }

    @Override
    public void save(Dept dept) {
        if (dept.getName() == null || dept.getName().isBlank()) {
            throw new IllegalArgumentException("部门名称不能为空");
        }
        mapper.insert(dept);
    }
}
~~~

### 2.4 Controller

~~~java
@RestController
@RequestMapping("/depts")
public class DeptController {
    private final DeptService service;

    public DeptController(DeptService service) {
        this.service = service;
    }

    @GetMapping
    public Result<List<Dept>> list() {
        return Result.success(service.list());
    }

    @PostMapping
    public Result<Void> save(@RequestBody Dept dept) {
        service.save(dept);
        return Result.success();
    }
}
~~~

参数接收方式要和请求格式匹配：

~~~java
@GetMapping("/{id}")
public Result<Dept> find(@PathVariable Long id) {
    return Result.success(service.find(id));
}

@GetMapping("/search")
public Result<List<Dept>> search(@RequestParam(required = false) String name) {
    return Result.success(service.search(name));
}

@PostMapping
public Result<Void> save(@Valid @RequestBody Dept dept) {
    service.save(dept);
    return Result.success();
}
~~~

`@RequestParam` 接收查询参数，`@PathVariable` 接收路径变量，`@RequestBody` 把 JSON 请求体转换为 Java 对象。使用 `@Valid` 后，实体类可以用 `@NotBlank`、`@Size` 等注解声明校验规则。

新增部门要校验名称非空、长度和唯一性；重复提交可能造成重复数据，应通过唯一索引、请求幂等设计或业务检查处理。新增可返回 201 或统一业务成功码，删除成功可返回 204 或统一空 data。异常时不要把数据库异常原文返回给前端。

### 2.5 请求流程

~~~mermaid
sequenceDiagram
    participant UI as Vue
    participant C as Controller
    participant S as Service
    participant M as Mapper
    participant DB as MySQL
    UI->>C: GET /depts
    C->>S: list()
    S->>M: findAll()
    M->>DB: SELECT
    DB-->>M: 数据
    M-->>C: 部门列表
    C-->>UI: JSON
~~~

### 2.6 修改和删除的实现要点

修改时先根据 ID 查询数据是否存在，再执行更新；删除时检查 Mapper 返回的受影响行数。业务层可以把“未找到记录”转换为明确的业务异常，由全局异常处理器统一返回。

~~~java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return ResponseEntity.noContent().build();
}
~~~

Controller 不应直接拼接 SQL，也不应把数据库异常原文返回给浏览器。

修改请求常见校验流程：先校验 JSON 格式和字段范围，再查询 ID 是否存在，最后执行更新并检查受影响行数。删除请求还要确认是否存在关联员工；如果不允许删除，应返回明确的业务错误，而不是让数据库报错。

### 2.7 接口测试与联调

接口测试时建议按“正常请求、缺少参数、参数格式错误、资源不存在、重复提交、数据库异常”逐项验证。Apifox 中应保存请求方法、URL、请求头、请求体和预期响应，便于前后端联调。

## 3. 日志技术 Logback

### 3.1 为什么不用 System.out.println

最原始的做法是 `System.out.println("查询部门，" + name)`。它能打印，但不能算日志：

| 对比项 | System.out.println | 日志框架（Logback） |
|---|---|---|
| 分级 | 没有级别概念，想看得细只能全打 | TRACE 到 ERROR 分级，可按级别过滤 |
| 开关 | 上线后想关掉只能删代码重新打包 | 改配置文件即可，不用动代码 |
| 输出位置 | 只能打控制台，容器里容易丢 | 控制台、文件、滚动文件可同时输出 |
| 格式 | 每处手写拼接，格式各不相同 | pattern 统一格式，含时间、线程、类名 |
| 性能 | 同步阻塞，且拼接无条件执行 | 支持异步 appender，占位符按需拼接 |
| 归档 | 没有 | RollingFileAppender 按时间或大小滚动，按天数保留 |
| 排查 | 没有时间、线程、类名、行号 | 默认带这些信息，能定位到具体代码行 |

还有一点容易被忽略：`println` 的字符串拼接是先算再输出，`log.debug("dept=" + list)` 无论当前级别是否输出 DEBUG，拼接都已经发生；数据量大时这笔开销很可观。

### 3.2 日志级别与使用场景

| 级别 | 高低顺序 | 使用场景 | 举例 |
|---|---|---|---|
| TRACE | 最低 | 极细粒度，逐步跟踪，只在本地排查时开 | 循环里每一步的中间值 |
| DEBUG | 次低 | 开发调试，看 SQL、参数和分支走向 | 打印 Mapper 执行的 SQL 和参数 |
| INFO | 中间 | 默认级别，记录关键业务流程节点 | 服务启动、接口入口出口、新增或删除成功 |
| WARN | 次高 | 有问题但能继续，需要关注 | 重试成功、参数被兜底值替换、缓存未命中 |
| ERROR | 最高 | 影响功能的异常，需要告警排查 | 数据库连不上、调用第三方接口失败 |

日志级别从低到高通常是 TRACE、DEBUG、INFO、WARN、ERROR。开发环境可以使用 DEBUG 查看 SQL 和参数，生产环境应避免输出敏感信息和过量日志。

级别是“门槛”：`<root level="info">` 表示 INFO 及以上才输出，DEBUG、TRACE 会被直接丢弃，所以不要把重要信息写在 DEBUG 里再指望生产环境能看到。反过来，可预期的业务失败（例如“部门名称已存在”）应该用 WARN 或业务异常表达，不要用 ERROR 制造告警噪音。

### 3.3 SLF4J 与 Logback 的关系

SLF4J 是门面（Facade），只提供 `Logger`、`LoggerFactory` 这些接口和统一的调用写法，本身不实现日志；Logback 是具体实现，负责真正把日志写到控制台和文件。业务代码只依赖 SLF4J 的接口，将来换成 Log4j2 时业务代码一行都不用改。Spring Boot 的 `spring-boot-starter` 已经间接引入了 `spring-boot-starter-logging`，里面就是 SLF4J 加 Logback，所以默认不需要自己再加依赖。

~~~mermaid
flowchart LR
    A[业务代码 log.info] --> B[SLF4J 接口]
    B --> C[Logback 实现]
    C --> D[ConsoleAppender]
    C --> E[RollingFileAppender]
~~~

使用 Logback 或 SLF4J 记录关键操作：

~~~java
private static final Logger log = LoggerFactory.getLogger(DeptController.class);

log.info("查询部门，name={}", name);
~~~

不要记录密码、完整令牌和数据库连接密码。

### 3.4 logback.xml 常见配置

Spring Boot 项目把 `logback.xml` 放在 `src/main/resources` 下就会被自动加载（想接入 Spring Boot 的 profile 能力，就命名为 `logback-spring.xml`）。一个够用的配置包含四部分：控制台 appender、文件 appender、按时间或大小滚动的 RollingFileAppender，以及用 `<root>`、`<logger>` 指定级别并引用 appender。

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- 控制台：开发时主要看它 -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
      <charset>UTF-8</charset>
    </encoder>
  </appender>

  <!-- 文件：按时间滚动，每天一个文件；同一天超过 10MB 再拆一个 -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
      <maxFileSize>10MB</maxFileSize>
      <maxHistory>30</maxHistory>
      <totalSizeCap>1GB</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
      <charset>UTF-8</charset>
    </encoder>
  </appender>

  <!-- 只给自己的包开 DEBUG，方便看 SQL 和参数 -->
  <logger name="com.example.mapper" level="debug"/>

  <!-- 根级别：INFO 及以上，同时输出到控制台和文件 -->
  <root level="info">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
  </root>
</configuration>
~~~

pattern 里的常用占位符：

| 占位符 | 含义 |
|---|---|
| `%d{yyyy-MM-dd HH:mm:ss.SSS}` | 时间，精确到毫秒 |
| `%thread` | 线程名，排查并发问题必备 |
| `%-5level` | 日志级别，左对齐占 5 位 |
| `%logger{36}` | logger 名（通常是类名），超过 36 个字符会缩写 |
| `%msg` | 日志内容 |
| `%n` | 换行符 |

几点注意：`<root>` 的级别是整体门槛，`<logger>` 只覆盖某个包或类；生产环境把 root 设成 `info`，需要看 SQL 时只给 `com.example.mapper` 开 `debug`；日志目录要确认应用有写权限，容器里建议挂载到宿主机，否则容器重建日志就没了。

日志格式应包含时间、级别、线程、类名和消息。业务关键操作可以记录操作者 ID、资源 ID 和结果，但必须脱敏密码、Token、身份证号等敏感字段。

### 3.5 Lombok 的 @Slf4j 与占位符

每个类都手写 `LoggerFactory.getLogger(Xxx.class)` 很啰嗦。在类上加 Lombok 的 `@Slf4j`，编译期会自动生成一个名为 `log` 的静态字段，等价于上一节那句 `getLogger`：

~~~java
@Slf4j
@RestController
@RequestMapping("/depts")
public class DeptController {
    private final DeptService service;

    public DeptController(DeptService service) {
        this.service = service;
    }

    @GetMapping
    public Result<List<Dept>> list(@RequestParam(required = false) String name) {
        // {} 是占位符：只有该级别真正输出时才会拼接字符串
        log.info("查询部门，name={}", name);
        List<Dept> depts = service.list(name);
        log.debug("查询到 {} 条部门记录", depts.size());
        return Result.success(depts);
    }
}
~~~

使用要点：

- 占位符用 `{}`，不要写成 `"name=" + name`；多个占位符按顺序对应，例如 `log.info("用户 {} 登录，部门 {}", username, deptName)`。
- 打印异常要把异常对象放在最后一个参数：`log.error("新增部门失败", e)`，这样才会输出完整堆栈；写成 `log.error(e.getMessage())` 只剩一行文字，定位不到出错位置。
- 占位符个数和参数个数要数清楚：占位符多了会原样输出 `{}`，参数多了虽然不报错，但日志内容对不上。
- 报“找不到符号 log”时，检查是否漏了 lombok 依赖，或 IDE 没装 Lombok 插件。

## 4. 工程搭建

### 4.1 三层结构与包命名

Spring Boot 项目按职责分三层，一个请求从外到内依次经过 Controller、Service、Mapper：

| 层 | 常用注解 | 职责 | 不应该做的事 |
|---|---|---|---|
| Controller | `@RestController`、`@RequestMapping` | 接收参数、参数校验、返回统一响应 | 写业务规则、拼 SQL |
| Service | `@Service` | 业务规则、事务边界、组合多次 Mapper 调用 | 直接操作 `HttpServletRequest` |
| Mapper | `@Mapper` | 只负责 SQL 和对象映射 | 写 if/else 业务判断 |

Controller 注入 Service 接口而不是实现类，Service 注入 Mapper 接口，这样替换实现或加代理（事务、AOP）时不用改调用方。

包结构按“层”划分，类名带上业务名：

~~~text
src/main/java/com/example
├── TliasApplication.java            // 启动类
├── controller/DeptController.java
├── service/DeptService.java         // 接口
├── service/impl/DeptServiceImpl.java
├── mapper/DeptMapper.java
├── pojo/Dept.java                   // 实体类
├── pojo/Result.java                 // 统一响应
└── exception/GlobalExceptionHandler.java

src/main/resources
├── application.yml
└── mapper/DeptMapper.xml            // 与 Mapper 接口对应
~~~

命名约定：Controller 以 `Controller` 结尾，Service 用接口加 `Impl` 实现类，Mapper 以 `Mapper` 结尾，实体类用表名对应的单数名词。

### 4.2 application.yml 常见配置

~~~yaml
server:
  port: 8080

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/tlias?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf8
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

mybatis:
  configuration:
    # 开启驼峰映射：create_time 自动映射到 createTime
    map-underscore-to-camel-case: true
  # Mapper XML 的位置，多个目录用英文逗号分隔
  mapper-locations: classpath:mapper/*.xml
  # 实体类所在包，XML 里的 resultType 可以只写类名
  type-aliases-package: com.example.pojo

logging:
  level:
    com.example.mapper: debug
~~~

逐项说明：

| 配置 | 作用 | 常见错误 |
|---|---|---|
| `server.port` | 应用端口 | 端口被占用会启动失败，换端口或结束占用进程 |
| `spring.datasource.url` | 数据库地址、库名和连接参数 | 漏写时区或时区写错，表现为连接失败或时间差 8 小时 |
| `spring.datasource.username/password` | 数据库账号 | 不要提交到 Git，用环境变量 `${DB_PASSWORD}` 或本地配置覆盖 |
| `spring.datasource.driver-class-name` | 驱动类 | MySQL 8 要用 `com.mysql.cj.jdbc.Driver` |
| `mybatis.configuration.map-underscore-to-camel-case` | 下划线转驼峰 | 忘了开会出现 `createTime` 为 null |
| `mybatis.mapper-locations` | XML 文件位置 | 路径写错会报 `Invalid bound statement` |
| `logging.level.xxx` | 指定包的日志级别 | 生产环境不要把 root 开成 debug |

配置优先级是：命令行参数 > 环境变量 > `application-{profile}.yml` > `application.yml`，所以本地可以用 `application-dev.yml` 覆盖数据库账号，且不提交到仓库。

### 4.3 启动类位置与组件扫描范围

`@SpringBootApplication` 是组合注解，包含 `@SpringBootConfiguration`、`@EnableAutoConfiguration` 和 `@ComponentScan`，其中 `@ComponentScan` 默认扫描“启动类所在包及其所有子包”。

- 启动类要放在最外层包，例如 `com.example.TliasApplication`，这样 `com.example.controller`、`com.example.service`、`com.example.mapper` 都在扫描范围内。
- 如果启动类放到 `com.example.controller` 下，`com.example.service` 就不会被扫描，启动时会报找不到 Bean。
- 确实需要跨包时，在启动类上用 `@ComponentScan({"com.example", "com.other"})` 显式扩大范围；Mapper 接口除了逐个加 `@Mapper`，也可以在启动类上加 `@MapperScan("com.example.mapper")` 统一扫描，两种方式选一种即可。
- 启动类只保留 `main` 方法，不要在启动类里写业务逻辑。

## 5. 数据封装成驼峰

数据库字段习惯用下划线（`create_time`、`dept_id`），Java 属性习惯用驼峰（`createTime`、`deptId`）。MyBatis 默认按“列名对应属性名”映射，什么都不做时 `create_time` 映射不到 `createTime`，查出来的 `createTime` 就是 `null`。

三种方案对比：

| 方案 | 写法 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| SQL 起别名 | `SELECT create_time AS createTime FROM dept` | 不依赖任何配置，单条 SQL 完全可控 | 每条 SQL 都要写，字段多时冗长、容易漏 | 只涉及个别字段，或不想改全局配置 |
| `resultMap` | `<resultMap id="deptMap" type="com.example.Dept">` 配 `<result column="create_time" property="createTime"/>` | 映射关系显式、可读，支持 `association`、`collection` 等嵌套映射 | 字段多时配置量大，改表结构要同步改两处 | 字段名与属性名差异大、需要嵌套映射 |
| 开启驼峰映射 | `mybatis.configuration.map-underscore-to-camel-case: true` | 一次配置全局生效，SQL 可以和表结构保持一致 | 只能解决下划线转驼峰，其它差异仍要 `resultMap` | 项目统一方案，最省事 |

resultMap 的两种写法：

~~~xml
<resultMap id="deptMap" type="com.example.Dept">
  <id column="id" property="id"/>
  <result column="create_time" property="createTime"/>
  <result column="update_time" property="updateTime"/>
</resultMap>

<select id="findAll" resultMap="deptMap">
  SELECT id, name, create_time, update_time FROM dept ORDER BY id DESC
</select>
~~~

~~~java
@Results({
    @Result(column = "id", property = "id"),
    @Result(column = "create_time", property = "createTime")
})
@Select("SELECT id, name, create_time, update_time FROM dept")
List<Dept> findAll();
~~~

注意以下几点：

- 第 2 章的 `@Select("SELECT id, name, create_time, update_time FROM dept ORDER BY id DESC")` 只有在开启驼峰映射或写了别名时，`Dept.createTime` 才有值；否则实体里是 `null`，前端看到时间为空，很容易误以为数据没保存。
- 开启驼峰映射后，`create_time` 能映射到 `createTime`，但自己起的别名要写对大小写：多表查询里应写 `d.name AS deptName`，不要写 `dept_name` 再指望自动转换（虽然能转，但和直接写驼峰别名混用容易出错）。
- 三种方案可以混用：整体开启驼峰映射，字段差异大的表用 `resultMap`，临时查询用别名。
- 用 `resultType` 时映射按列名和属性名匹配，用 `resultMap` 时完全按配置匹配，两者不要一起写。

## 6. 前后端联调与 nginx 反向代理

前后端分离开发时，前端页面和后端接口通常不同源（端口或主机不同），浏览器直接请求后端会被同源策略拦下。开发阶段常用 nginx 反向代理统一入口：浏览器只访问 nginx，nginx 再把不同前缀的请求转发给不同服务。

~~~mermaid
flowchart LR
    B[浏览器 http://localhost:90] --> N[nginx :90]
    N -->|/api/ 转发| A[Spring Boot :8080]
    N -->|其他路径| S[前端静态资源 dist]
    A --> D[(MySQL)]
~~~

nginx 最小配置（`conf/nginx.conf` 的 `server` 段）：

~~~text
server {
    listen       90;
    server_name  localhost;

    # 前端打包后的静态资源
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }

    # 接口请求统一以 /api/ 开头，转发给后端
    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
    }
}
~~~

几点说明：

- `location /api/` 后面的斜杠和 `proxy_pass` 结尾的斜杠要成对写：`proxy_pass http://127.0.0.1:8080/;` 会把 `/api/depts` 转发成 `http://127.0.0.1:8080/depts`，从而去掉 `/api` 前缀；如果 `proxy_pass` 后面不写斜杠，前缀会被一起带过去变成 `/api/depts`，后端匹配不到接口。
- 改完配置先 `nginx -t` 检查语法，再 `nginx -s reload` 让其生效，不需要重启进程。
- 统一入口的好处：前端只需要配置一个基础地址 `/api`，切换测试或生产环境时只改 nginx，不用重新打包前端代码。
- 规避跨域的原理：浏览器眼里所有请求都是发给同源的 nginx，跨域限制不成立；真正的跨源请求发生在 nginx 和 Spring Boot 之间，而服务器之间的 HTTP 调用不受同源策略约束。
- 如果不想用 nginx，也可以在后端配置 CORS，但要同时处理允许的来源、请求方法和预检请求，联调阶段不如统一入口省心；生产环境一般由 nginx 或网关承担这一层。

## 7. 本章总结

- 部门管理实现查询、新增、修改、删除。
- Apifox 用于接口测试，Vue 用于联调。
- Controller、Service、Mapper 分层使代码更容易维护。
- 统一响应只保留 `code`、`msg`、`data` 三个字段，成功时 `code` 为 1。
- 工程搭建记住三件事：三层结构加规范包名、`application.yml` 配好数据源和 MyBatis、启动类放在最外层包。
- 日志用 SLF4J 门面加 Logback 实现，`logback.xml` 控制级别和滚动策略，业务代码用 `@Slf4j` 和 `{}` 占位符。
- 下划线字段映射到驼峰属性可以起别名、用 `resultMap` 或开启驼峰映射，项目里统一一种主方案。
- 联调阶段用 nginx 反向代理统一入口，既能一套地址访问前后端，也顺带绕开了浏览器的跨域限制。
