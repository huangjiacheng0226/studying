# 09 后端 Web 实战（员工管理：新增、事务和文件上传）

## 1. 新增员工

### 1.1 业务流程

员工基本信息和工作经历需要同时保存，属于一个业务整体。

新增员工接口通常接收基本信息和工作经历集合。处理顺序是先校验请求，再插入员工主表；数据库回填主键后，把主键作为工作经历表的外键批量插入。

~~~mermaid
flowchart TD
    A[接收员工 JSON] --> B[参数校验]
    B --> C[插入 emp]
    C --> D[取得员工主键]
    D --> E[批量插入工作经历]
    E --> F[提交事务]
    C -->|失败| X[回滚]
    E -->|失败| X
~~~

### 1.2 事务代码

~~~java
@Transactional(rollbackFor = Exception.class)
public void add(Emp emp) {
    empMapper.insert(emp); // 插入后回填主键
    if (emp.getExprList() != null) {
        exprMapper.insertBatch(emp.getId(), emp.getExprList());
    }
}
~~~

rollbackFor 明确哪些异常触发回滚；propagation 控制事务传播，常见 REQUIRED 表示加入现有事务或新建事务。

默认情况下，Spring 对运行时异常回滚，受检异常不一定自动回滚，因此跨表写入常使用 `rollbackFor = Exception.class`。常见传播行为：`REQUIRED` 加入当前事务或新建事务，`REQUIRES_NEW` 暂停当前事务并新建事务，`SUPPORTS` 有事务就加入、没有就非事务执行。

@Transactional 应放在 Service 的 public 方法上，并通过 Spring 代理调用；同类内部直接调用可能绕过代理。事务内不要执行耗时的远程 OSS 上传，避免长时间占用数据库连接。

跨表写入失败时必须回滚已完成的写操作，否则会出现只有员工没有经历、或只有部分经历的脏数据。批量插入应确认集合为空时不会生成非法 SQL。

事务生效的前提是方法由 Spring 代理对象调用、数据库引擎支持事务（MySQL 常用 InnoDB），并且异常没有被方法内部吞掉。不要在事务方法中捕获异常后只打印日志却继续返回成功，否则 Spring 无法感知失败并回滚。

### 1.3 主键回填：@Options

MyBatis 的 `insert` 默认只返回影响行数，不会把数据库生成的自增主键写回 Java 对象，因此插入后调用 `emp.getId()` 可能得到 `null`，工作经历表就没有外键可用。

`@Options(useGeneratedKeys = true, keyProperty = "id")` 的作用是：让 MyBatis 通过 JDBC 的 `getGeneratedKeys()` 取回数据库生成的主键，并回填到实体对象的指定属性上。

| 属性 | 含义 | 常见取值 |
|---|---|---|
| `useGeneratedKeys` | 是否使用数据库生成的主键 | `true` |
| `keyProperty` | 回填到实体类的哪个属性 | `id` |
| `keyColumn` | 主键对应的数据库列名 | 属性名与列名不一致时填写 |

注解写法：

~~~java
@Mapper
public interface EmpMapper {
    @Options(useGeneratedKeys = true, keyProperty = "id")
    @Insert("insert into emp(username, name, gender, job, dept_id, create_time) " +
            "values (#{username}, #{name}, #{gender}, #{job}, #{deptId}, #{createTime})")
    void insert(Emp emp);
}
~~~

XML 映射文件中把同样的属性写在 `<insert>` 标签上：

~~~xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into emp(username, name, gender, job, dept_id, create_time)
    values (#{username}, #{name}, #{gender}, #{job}, #{deptId}, #{createTime})
</insert>
~~~

只有插入语句才有回填的意义，且必须依赖数据库的自增主键或序列。取值回填后可以立刻用于子表插入，这也是"先插主表、再插工作经历"能成立的前提。

### 1.4 `<foreach>` 批量插入

术语：`<foreach>` 是 MyBatis 的动态 SQL 标签，用于遍历集合并把每次迭代拼接成一段 SQL。如果工作经历有 10 条，循环调用单条插入会产生 10 次数据库往返，用 `<foreach>` 可以合并成一条 `insert`。

| 属性 | 含义 |
|---|---|
| `collection` | 要遍历的集合，取值为方法参数名或实体属性名 |
| `item` | 每次迭代的元素变量名 |
| `index` | 迭代下标（List 为序号，Map 为 key） |
| `separator` | 元素之间的分隔符 |
| `open` / `close` | 在拼接结果前后添加的符号 |

方法有多个参数时必须用 `@Param` 命名，否则 MyBatis 只能用 `arg0`、`param1` 之类的位置名，`collection` 将无法正确对应：

~~~java
void insertBatch(@Param("empId") Long empId, @Param("exprList") List<EmpExpr> exprList);
~~~

~~~xml
<insert id="insertBatch">
    insert into emp_expr(emp_id, begin, end, company, job)
    values
    <foreach collection="exprList" item="expr" separator=",">
        (#{empId}, #{expr.begin}, #{expr.end}, #{expr.company}, #{expr.job})
    </foreach>
</insert>
~~~

与循环单条插入相比：

| 维度 | 循环单条插入 | `foreach` 批量插入 |
|---|---|---|
| 数据库往返次数 | N 次 | 1 次 |
| SQL 解析次数 | N 次 | 1 次 |
| 网络开销 | 随记录数线性增长 | 一次传输 |
| 失败影响范围 | 可能只失败其中一条 | 整条 SQL 失败，通常配合事务回滚 |
| 适合数据量 | 少量数据 | 中小批量，建议分批（如每 500 条） |

单条 SQL 过大可能超过数据库的 `max_allowed_packet` 限制，所以超大集合要分批提交；集合为空时不要生成只有 `values` 没有值的非法 SQL。

### 1.5 事务传播行为：REQUIRED 与 REQUIRES_NEW

术语：事务传播行为（propagation）回答的是"一个事务方法调用另一个事务方法时，被调用方是加入已有事务，还是另起一个事务"。它解决的是事务边界问题，不是并发隔离问题。

| 传播行为 | 有外层事务时 | 没有外层事务时 | 典型场景 |
|---|---|---|---|
| `REQUIRED`（默认） | 加入外层事务，成败与共 | 新建事务 | 绝大多数业务写入 |
| `REQUIRES_NEW` | 挂起外层事务，新建独立事务 | 新建事务 | 日志、审计等必须独立提交的记录 |
| `SUPPORTS` | 加入外层事务 | 以非事务方式执行 | 可选事务的查询 |
| `NOT_SUPPORTED` | 挂起外层事务，非事务执行 | 非事务执行 | 长耗时的导入或调用 |
| `MANDATORY` | 加入外层事务 | 直接抛异常 | 强制要求必须在事务中调用 |
| `NEVER` | 抛异常 | 非事务执行 | 禁止在事务中调用 |
| `NESTED` | 以保存点方式嵌套，内层可单独回滚 | 新建事务 | 需要部分回滚的场景 |

`REQUIRED` 与 `REQUIRES_NEW` 的核心差别只有一句话：前者和调用方是同一个事务，后者是两个事务（外层被挂起，内层独立提交或回滚）。

用"添加员工时同时写操作日志 `emp_log`，日志失败不能影响员工插入"来说明差别。日志方法与员工新增分成两个 Bean：

~~~java
@Service
public class EmpLogServiceImpl implements EmpLogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void insertLog(String message) {
        empLogMapper.insert(message);
    }
}
~~~

~~~java
@Service
public class EmpServiceImpl implements EmpService {
    @Autowired
    private EmpLogService empLogService;

    @Transactional(rollbackFor = Exception.class)
    public void add(Emp emp) {
        empMapper.insert(emp);
        try {
            empLogService.insertLog("新增员工 " + emp.getId());
        } catch (Exception e) {
            log.warn("写入操作日志失败", e); // 日志失败不影响员工插入
        }
    }
}
~~~

两种传播行为下的实际结果：

| 场景 | `insertLog` 用 `REQUIRED` | `insertLog` 用 `REQUIRES_NEW` |
|---|---|---|
| 日志插入成功、员工插入成功 | 同一事务一次提交 | T1 提交员工，T2 提交日志 |
| 日志插入失败 | 事务被标记 rollback-only，员工插入也会回滚 | 只回滚日志事务 T2，T1 仍可提交 |
| 外层员工插入失败 | 日志一起回滚 | 日志已独立提交，可能留下"有日志无业务"的记录 |
| 数据库连接 | 复用外层连接 | 额外占用一个连接 |

因此 `insertLog` 要用 `REQUIRES_NEW`，让日志拥有自己的事务边界。但要注意：`REQUIRES_NEW` 只解决事务边界，异常仍会继续上抛，如果外层不 catch，`add` 照样回滚。所以"独立事务 + 外层捕获异常"要一起使用。

~~~mermaid
sequenceDiagram
    participant C as Controller
    participant A as EmpServiceImpl.add
    participant L as EmpLogServiceImpl.insertLog
    participant DB as MySQL
    C->>A: 调用 add
    A->>DB: 开启事务 T1，插入 emp
    A->>L: 写入操作日志
    L->>DB: 挂起 T1，开启事务 T2
    L->>DB: 插入 emp_log
    L->>DB: 提交或回滚 T2
    L-->>A: 返回结果（异常由外层决定是否捕获）
    A->>DB: 提交 T1
~~~

`REQUIRES_NEW` 会额外占用一个数据库连接，外层连接在挂起期间仍然被持有，并发高时容易耗尽连接池。日志量很大时应考虑异步写库或消息队列，这与第 13 章"操作日志的事务取舍"是同一个权衡。

### 1.6 自调用导致传播行为失效

`@Transactional` 依赖 Spring 代理：外部请求先进入代理对象，代理负责开启、挂起、提交或回滚事务，然后才调用目标方法。在同一个类中用 `this.method()` 调用另一个方法，绕过了代理，事务注解不会被解析，传播行为自然失效。

~~~java
@Service
public class EmpServiceImpl implements EmpService {
    @Transactional(rollbackFor = Exception.class)
    public void add(Emp emp) {
        empMapper.insert(emp);
        insertLog("新增员工 " + emp.getId()); // 相当于 this.insertLog()，代理不生效
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void insertLog(String message) {
        empLogMapper.insert(message);
    }
}
~~~

上面的写法里 `insertLog` 的 `REQUIRES_NEW` 不会生效，日志会跟随外层事务一起提交或回滚，与写在同一个事务里没有区别。

~~~mermaid
flowchart TD
    A[Controller 调用 add] --> B[Spring 代理]
    B --> C[开启事务 T1]
    C --> D[目标对象 add]
    D --> E{调用 insertLog 的方式}
    E -->|this.insertLog| F[本类内部调用，不经过代理]
    E -->|empLogService.insertLog| G[代理拦截，按 REQUIRES_NEW 开启 T2]
    F --> H[日志加入 T1，随 T1 一起提交或回滚]
    G --> I[日志在 T2 独立提交]
~~~

| 方案 | 写法 | 说明 |
|---|---|---|
| 拆到另一个 Bean | 独立的 `EmpLogService` | 最清晰，天然经过代理，推荐首选 |
| 注入自身代理 | `@Autowired private EmpService self;` 后 `self.insertLog(...)` | 改动小；注意类实现接口时按接口类型注入 |
| 自身代理加懒加载 | `@Lazy @Autowired private EmpService self;` | 可缓解构造期的循环依赖报错 |
| `AopContext.currentProxy()` | 需开启 `@EnableAspectJAutoProxy(exposeProxy = true)` | 侵入业务代码，配置漏写就报错，不推荐 |
| 从容器取 Bean | `applicationContext.getBean(EmpService.class)` | 可行但让业务代码依赖容器 |

排查口诀：事务或切面不生效时，先看是不是自己调自己，再看方法是否 `public`、是否被 `final`/`static` 修饰、异常是否被方法内部吞掉。

## 2. 文件上传

### 2.1 本地存储示例

~~~java
@PostMapping("/upload")
public Result<String> upload(@RequestParam MultipartFile image) throws IOException {
    if (image.isEmpty()) {
        throw new IllegalArgumentException("文件不能为空");
    }
    String ext = FilenameUtils.getExtension(image.getOriginalFilename());
    String fileName = UUID.randomUUID() + "." + ext;
    Path target = Paths.get("uploads", fileName);

    Files.createDirectories(target.getParent());
    image.transferTo(target);
    return Result.success("/uploads/" + fileName);
}
~~~

上传接口接收 `MultipartFile`，文件内容通过请求体传输，表单的 `Content-Type` 必须是 `multipart/form-data`。保存成功后通常只返回文件访问地址或业务文件 ID，不直接返回服务器绝对路径。

### 2.2 安全规范

- 限制文件大小、扩展名和 MIME 类型。
- 使用随机文件名，避免覆盖已有文件。
- 不信任原始文件名，防止路径穿越。
- 生产环境使用 OSS 等对象存储，并控制访问权限。

Spring Boot 可通过 spring.servlet.multipart.max-file-size 和 max-request-size 限制上传大小。上传接口应限制 Content-Type、扩展名、文件头，并考虑病毒扫描和访问权限。

文件名应由服务器生成，不能直接使用用户上传的原始文件名。扩展名校验不能代替文件头校验；还应防止路径穿越、脚本文件上传和未授权访问。多实例部署时，本地磁盘文件不会自动共享，应使用 OSS 或其他对象存储。

对象存储的一般流程是：申请临时访问凭证或使用服务端密钥、上传文件、保存对象地址或对象键、按权限返回访问地址。数据库事务和远程文件上传通常不放在同一个长事务中；上传失败时要清理已上传的孤儿文件。

文件上传接口的前端表单必须设置 `enctype="multipart/form-data"`。后端可用 `@RequestParam("image") MultipartFile image` 接收文件，用 `image.getSize()`、`getContentType()` 和文件头做校验。下载或访问文件时还要检查当前用户是否有权限。

### 2.3 MultipartFile 的常用方法

术语：`MultipartFile` 是 Spring 对 `multipart/form-data` 请求中单个文件的封装，位于 `org.springframework.web.multipart` 包；Spring Boot 默认使用 Servlet 容器的 multipart 解析器实现它。

| 方法 | 作用 | 注意点 |
|---|---|---|
| `getOriginalFilename()` | 客户端上传时的原始文件名 | 用户可伪造且可能带路径，只用来取扩展名 |
| `getSize()` | 文件的字节数 | 空文件为 0，注意单位换算 |
| `isEmpty()` | 是否没有内容 | 判断"前端没有选文件"更直观 |
| `getContentType()` | 客户端声明的 MIME 类型 | 由客户端提供，不能作为唯一校验依据 |
| `getBytes()` | 一次性读成字节数组 | 大文件会占用大量堆内存 |
| `getInputStream()` | 得到输入流 | 适合流式写盘或上传到 OSS |
| `transferTo(File/Path)` | 把内容写入目标文件 | 相对路径基于容器临时目录，建议用绝对路径 |
| `getResource()` | 转换成 `Resource` | 便于统一用资源 API 处理 |

~~~java
@PostMapping("/upload")
public Result<String> upload(MultipartFile file) throws IOException {
    if (file.isEmpty()) {
        throw new BizException("请选择文件");
    }
    if (file.getSize() > 2 * 1024 * 1024) {
        throw new BizException("文件不能超过 2MB");
    }
    String name = UUID.randomUUID() + ".jpg";              // 文件名由服务器生成
    Path target = Paths.get("/data/uploads").resolve(name); // 绝对路径
    file.transferTo(target);
    return Result.success("/uploads/" + name);
}
~~~

两个容易踩的坑：

- 同一个 `MultipartFile` 的内容只能读取一次。先调用 `getBytes()` 再调用 `transferTo()`，在部分实现上会因为流已消费而失败；需要多次使用时，先把字节数组保存下来，或用 `Files.copy(file.getInputStream(), target)` 自行写盘。
- `transferTo(File)` 传相对路径时，基准目录是容器的临时目录而不是项目目录，容易出现"文件不知道存到哪去了"，因此统一使用绝对路径。

### 2.4 上传大小限制配置

Spring Boot 用两个配置项限制 multipart 请求，默认值很小，不改配置时稍大的图片就会被拒绝：

| 配置项 | 作用 | 默认值 |
|---|---|---|
| `spring.servlet.multipart.max-file-size` | 单个文件的最大字节数 | 1MB |
| `spring.servlet.multipart.max-request-size` | 整个 multipart 请求的最大字节数（所有文件与表单字段之和） | 10MB |
| `spring.servlet.multipart.file-size-threshold` | 超过该阈值就写到磁盘，小于则放内存 | 0 |
| `spring.servlet.multipart.location` | 临时文件目录 | 容器默认 |

~~~yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 100MB
~~~

等价的 properties 写法是 `spring.servlet.multipart.max-file-size=10MB`，取值支持 `KB`、`MB`、`GB` 单位（不写单位时按字节理解）。

超过限制时 Spring 抛出 `MaxUploadSizeExceededException`，应在全局异常处理器中单独捕获，返回可读提示而不是 500：

~~~java
@ExceptionHandler(MaxUploadSizeExceededException.class)
public Result<Void> handleUploadSize(MaxUploadSizeExceededException ex) {
    return Result.error("上传文件过大，请压缩后再试");
}
~~~

也可以用 `MultipartConfigElement`、`MultipartConfigFactory` 以编程方式配置，但不要与配置文件同时设置，避免两处值互相覆盖难以排查。另外，Nginx 的 `client_max_body_size`、网关和容器通常也有各自的请求体上限，"还没进 Controller 就失败"时应逐层检查。

### 2.5 文件名唯一化与扩展名校验

要点先列清楚，再给代码：

- 文件名一律由服务器生成，用 `UUID.randomUUID()` 或雪花 ID，避免同名覆盖和路径穿越。
- 扩展名用白名单校验，不要用黑名单（黑名单总会漏）。
- 从原始文件名取出扩展名后还要用正则限制字符集，防止 `../`、`%00` 之类构造出非法路径。
- 扩展名不能代替文件头校验：常见图片可校验 Magic Number（JPEG 以 `FF D8 FF` 开头，PNG 以 `89 50 4E 47` 开头）。

~~~java
private static final Set<String> ALLOWED = Set.of("jpg", "jpeg", "png", "gif");

public String buildObjectName(MultipartFile file) {
    String original = file.getOriginalFilename();
    if (original == null || !original.contains(".")) {
        throw new BizException("文件名不合法");
    }
    String ext = original.substring(original.lastIndexOf('.') + 1).toLowerCase();
    if (!ext.matches("[a-z0-9]{1,10}") || !ALLOWED.contains(ext)) {
        throw new BizException("只支持 jpg、png、gif 图片");
    }
    return "emp/" + LocalDate.now() + "/" + UUID.randomUUID() + "." + ext;
}
~~~

对象名按日期分目录（如 `emp/2025-01-01/...`）可以让单个目录下的对象数量不至于过多，也方便按时间清理历史文件。返回给前端的地址由服务端拼接，不要回显服务器绝对路径，更不要把密钥拼进 URL。

### 2.6 阿里云 OSS 文件上传

#### 2.6.1 为什么不用本地存储

本地存储（把文件 `transferTo` 到本机磁盘）适合单机学习，放到真实部署环境会立刻暴露下面的问题：

| 问题 | 具体表现 |
|---|---|
| 磁盘容量有限 | 图片、视频很快占满单机磁盘，扩容往往要停机加盘或换机器 |
| 多台服务器不共享 | 负载均衡后文件只落在其中一台，下次请求被分到别的实例就是 404 |
| 扩容与备份麻烦 | 每台机器都要挂盘、同步、定时备份，运维成本随实例数线性增长 |
| 迁移要搬文件 | 换服务器或换机房时，必须把历史文件全部拷贝过去 |
| 容易丢文件 | 容器重建、临时目录清理、误删目录都会直接丢数据 |
| 访问速度受限 | 所有下载都走应用服务器带宽，图片一多就拖垮接口 |

#### 2.6.2 OSS 的优势

术语：OSS（Object Storage Service，对象存储服务）是云厂商提供的对象存储，用"Bucket（存储空间）+ Object（对象）"组织数据。对象通过唯一的对象名（key）定位，并通过 HTTP(S) 访问；key 中的 `/` 只是视觉上模拟目录，并不存在真实文件夹。

| 优势 | 说明 |
|---|---|
| 海量存储 | 容量近似无限，不用预估磁盘；多实例天然共享同一份文件 |
| 按量付费 | 按存储量、请求次数、流量计费，学习和中小项目成本很低 |
| CDN 加速 | 可接入 CDN 就近分发，下载速度不再受源站带宽限制 |
| 权限控制 | 支持 RAM 子账号、Bucket 读写权限、STS 临时凭证、预签名 URL |
| 可靠性与治理 | 服务端多副本存储，支持版本控制、生命周期规则、跨区域复制 |

与本地磁盘横向对比：

| 维度 | 本地磁盘 | 阿里云 OSS |
|---|---|---|
| 多实例共享 | 不支持，需额外同步或挂共享盘 | 原生支持 |
| 扩容 | 加盘或换机器 | 无感扩容 |
| 备份与容灾 | 自己写脚本、自己做副本 | 服务端多副本，可配置版本与备份 |
| 访问速度 | 受源站带宽限制 | 可结合 CDN 加速 |
| 权限粒度 | 依赖文件系统权限 | 账号、Bucket、对象、临时凭证多级控制 |
| 成本结构 | 一次性硬件投入 | 按量付费，小规模更便宜 |
| 迁移成本 | 需要搬运历史文件 | 换 Bucket 只需要改配置 |

结论：本地磁盘用来理解"文件上传"这条链路，生产环境用 OSS 这类对象存储解决共享、扩容和备份问题。

#### 2.6.3 Java SDK 上传的关键步骤

先明确四个核心配置，缺一个都连不上 OSS：

| 配置项 | 含义 | 示例 |
|---|---|---|
| `endpoint` | 服务访问域名，决定了数据所在的地域 | `https://oss-cn-hangzhou.aliyuncs.com` |
| `accessKeyId` | 访问身份标识 | `${OSS_ACCESS_KEY_ID}` |
| `accessKeySecret` | 访问密钥 | `${OSS_ACCESS_KEY_SECRET}` |
| `bucketName` | 存储空间名称，全局唯一 | `${OSS_BUCKET_NAME}` |

相关术语：

- Bucket：存储空间，创建时要选择地域和读写权限，名称全局唯一。
- Object：对象，相当于一个文件，用对象名（key）标识。
- Endpoint：访问域名，例如 `oss-cn-hangzhou.aliyuncs.com`，其中包含地域信息。
- `putObject`：上传接口，把本地文件或输入流写入指定 Bucket 的指定 key。

上传步骤：

1. 引入 OSS SDK 依赖。
2. 在 `application.yaml` 中配置四个参数，密钥用环境变量占位，绝不写进仓库。
3. 创建 `OSSClient`，或按 3.3 节的方式注册成单例 Bean 复用。
4. 调用 `putObject(bucketName, objectKey, inputStream)` 上传文件。
5. 拼接可访问 URL，把 URL 或对象 key 保存到数据库。
6. 用完关闭客户端，释放连接。

~~~xml
<dependency>
    <groupId>com.aliyun.oss</groupId>
    <artifactId>aliyun-sdk-oss</artifactId>
    <version>3.17.4</version>
</dependency>
~~~

版本号以官方文档和项目实际情况为准，不要机械照搬资料中的旧版本。

~~~java
@Component
public class AliyunOSSOperator {
    private final OssProperties properties; // 配置绑定见 3.3 节

    public AliyunOSSOperator(OssProperties properties) {
        this.properties = properties;
    }

    public String upload(MultipartFile file, String objectName) throws IOException {
        OSS ossClient = new OSSClientBuilder().build(
                properties.getEndpoint(),
                properties.getAccessKeyId(),
                properties.getAccessKeySecret());
        try (InputStream in = file.getInputStream()) {
            ossClient.putObject(properties.getBucketName(), objectName, in);
        } finally {
            ossClient.shutdown();
        }
        return url(objectName);
    }

    private String url(String objectName) {
        String host = URI.create(properties.getEndpoint()).getHost();
        return "https://" + properties.getBucketName() + "." + host + "/" + objectName;
    }
}
~~~

上传成功后拼接出的地址形如 `https://${OSS_BUCKET_NAME}.oss-cn-hangzhou.aliyuncs.com/emp/2025-01-01/xxx.jpg`。如果 Bucket 是私有读，这个地址不能直接访问，需要改用 `generatePresignedUrl` 生成带签名的限时地址。

~~~mermaid
flowchart TD
    A[前端表单 multipart/form-data] --> B[Controller 接收 MultipartFile]
    B --> C[校验大小、扩展名、文件头]
    C --> D[生成唯一对象名]
    D --> E[AliyunOSSOperator.upload]
    E --> F[OSSClient.putObject]
    F -->|成功| G[拼接可访问 URL]
    G --> H[把 URL 写入员工记录]
    F -->|失败| I[抛异常，不写数据库]
    H --> J[事务提交]
~~~

安全与运维注意点：

- 密钥只放在环境变量、配置中心或密钥管理系统，仓库中一律使用 `${OSS_ACCESS_KEY_ID}`、`${OSS_ACCESS_KEY_SECRET}` 这类占位符。
- 生产环境优先使用 RAM 子账号（只授予目标 Bucket 必要权限）或 STS 临时凭证，不要使用主账号 AccessKey，更不要把 Bucket 设为公共读写。
- 大文件使用分片上传或断点续传接口，不要把整个文件读进内存。
- 远程上传不要放在长事务里：先生成对象名、上传成功后再写数据库；上传成功但入库失败时要有清理孤儿文件的补偿逻辑。
- 不同 SDK 版本的类名与包名可能变化，实施时以官方文档为准。

延伸阅读：阿里云 OSS 官方文档 `https://help.aliyun.com/zh/oss/`。

## 3. Spring 配置注入

配置写在 `application.yaml` 里，代码还要能读到它们。Spring 提供两种读取方式：`@Value` 和 `@ConfigurationProperties`。第 14B 章从自动配置的角度做过对比，这里从"怎么写、什么时候用哪个"的角度展开。

### 3.1 用 @Value 读取单个配置

`@Value` 通过占位符 `${key}` 把配置值注入到字段、构造器参数或方法参数上，适合少量零散的简单值。

~~~yaml
aliyun:
  oss:
    endpoint: https://oss-cn-hangzhou.aliyuncs.com
    bucket-name: ${OSS_BUCKET_NAME}
~~~

~~~java
@Component
public class OssClientConfig {
    @Value("${aliyun.oss.endpoint}")
    private String endpoint;

    @Value("${aliyun.oss.bucket-name}")
    private String bucketName;

    @Value("${aliyun.oss.dir:emp/}") // 冒号后面是默认值
    private String dir;
}
~~~

`@Value` 的限制：

| 限制 | 说明 |
|---|---|
| 只适合简单值 | 绑定 `List`、`Map`、嵌套对象要写 SpEL 表达式，可读性差 |
| 不支持松散绑定 | `${aliyun.oss.access-key-id}` 必须按配置中的写法精确匹配，不能写成 `accessKeyId` |
| 不能批量绑定 | 每个属性都要单独写一行，配置项一多就容易漏 |
| 依赖容器创建对象 | 自己 `new` 出来的对象不会被注入，必须交给 Spring 管理 |
| 静态字段无效 | `static` 字段不能直接注入，需要在 setter 中赋值 |
| 默认值语法特殊 | 默认值写在冒号后，形如 `${key:default}` |

### 3.2 用 @ConfigurationProperties 批量绑定

术语：`@ConfigurationProperties` 把配置文件中某个前缀下的一整组属性绑定到一个 JavaBean。`prefix` 指定前缀，字段名按松散绑定规则与配置项匹配。

它支持批量绑定、类型安全、嵌套对象与列表，也便于集中管理和校验。

| 对比项 | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| 绑定范围 | 单个属性 | 一个前缀下的整组属性 |
| 复杂类型 | 需要 SpEL，写法繁琐 | 直接支持嵌套对象、`List`、`Map` |
| 松散绑定 | 不支持 | 支持 `access-key-id`、`accessKeyId`、`access_key_id` 等写法 |
| 默认值 | 写在 `${key:default}` 中 | 在字段上直接给默认值 |
| 数据校验 | 需要额外处理 | 配合 `@Validated` 使用 JSR-303 注解 |
| 适用场景 | 零散的一两个值 | 一组有关联的配置 |

### 3.3 注册方式与完整示例

`@ConfigurationProperties` 只是声明绑定规则，属性类还必须被容器管理，常见三种注册方式：

| 方式 | 写法 | 适用场景 |
|---|---|---|
| 加 `@Component` | 类上同时写 `@ConfigurationProperties` 和 `@Component` | 业务项目里最简单直接 |
| `@EnableConfigurationProperties` | 在配置类或启动类上引入属性类 | 属性类来自第三方包、不能加注解时 |
| `@ConfigurationPropertiesScan` | 启动类上开启扫描 | 属性类较多且位于同一包下 |

`application.yaml` 片段：

~~~yaml
aliyun:
  oss:
    endpoint: https://oss-cn-hangzhou.aliyuncs.com
    access-key-id: ${OSS_ACCESS_KEY_ID}
    access-key-secret: ${OSS_ACCESS_KEY_SECRET}
    bucket-name: ${OSS_BUCKET_NAME}
    dir: emp/
~~~

对应的属性类：

~~~java
@Component
@ConfigurationProperties(prefix = "aliyun.oss")
public class OssProperties {
    private String endpoint;
    private String accessKeyId;
    private String accessKeySecret;
    private String bucketName;
    private String dir = "emp/";

    // 省略 getter/setter
}
~~~

要点：

- 必须有 getter/setter，或在类上使用构造器绑定；只有字段无法完成绑定。
- 字段名用驼峰，配置项可以写 `access-key-id`，松散绑定会自动对应到 `accessKeyId`。
- 密钥写成 `${OSS_ACCESS_KEY_ID}` 这类占位符，从环境变量读取；不要在 yaml 中粘贴真实 AccessKey。
- 需要校验时加上 `@Validated` 和 `@NotBlank` 等注解，配置缺失时启动即失败，比运行到上传时才报错更容易定位。

~~~mermaid
flowchart LR
    A[application.yaml] --> B[Environment 合并配置源]
    C[环境变量 OSS_ACCESS_KEY_ID] --> B
    B --> D{绑定方式}
    D -->|@Value| E[逐个属性注入]
    D -->|@ConfigurationProperties| F[整组绑定到 OssProperties]
    F --> G[AliyunOSSOperator 使用]
~~~

小结：零散的简单值用 `@Value`，一组有层次的配置（例如给 OSS 工具类使用的那四个参数）用 `@ConfigurationProperties`。

## 4. 事务四大特性

| 特性 | 含义 |
|---|---|
| 原子性 | 全部成功或全部失败 |
| 一致性 | 数据满足业务和约束 |
| 隔离性 | 并发事务互不产生错误影响 |
| 持久性 | 提交结果能够保存 |

隔离性通过数据库隔离级别实现。常见并发问题包括脏读、不可重复读和幻读；初学阶段先理解“一个事务看到的数据是否会被另一个事务中途修改”，再学习不同隔离级别的差异。

## 5. 本章总结

- 跨表写入必须放在同一事务，主键回填和批量插入是"先主后子"的落地手段。
- 事务传播行为决定事务边界：日志这类附属记录用 `REQUIRES_NEW` 独立提交，并注意自调用会让注解失效。
- 文件上传要校验类型、大小、文件名和权限。
- 本地磁盘适合学习，OSS 更适合多实例部署；OSS 的四个参数用 `@ConfigurationProperties` 成组绑定，密钥只写占位符。
