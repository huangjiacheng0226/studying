# 10 后端 Web 实战（员工管理：修改、删除、异常和统计）

## 1. 修改与删除

### 1.1 接口流程

~~~text
GET    /emps/{id}  -> 查询回显
PUT    /emps       -> 提交修改
DELETE /emps/{id}  -> 删除员工
~~~

修改通常分为查询回显和提交更新两个阶段。删除前要考虑工作经历等关联数据和重复请求。

修改接口一般先通过 ID 查询详情，前端回显表单后提交完整或部分字段。更新时必须校验 ID 是否存在、字段是否合法，并只修改允许修改的字段。删除员工前要明确工作经历等子表记录是级联删除、逻辑删除还是禁止删除。

删除接口应检查受影响行数，删除不存在的 ID 时返回明确的业务结果。重复点击删除按钮不会产生额外副作用，但前端仍应在请求发送后禁用按钮，避免重复请求。

更新操作应使用白名单字段，避免客户端提交 `createTime`、`password` 或权限字段导致越权修改。数据库层可使用 `WHERE id = ?`，Service 层检查返回行数为 1；若为 0，说明记录不存在或已被其他请求修改。

修改接口参数可以使用路径变量和 JSON 请求体：

~~~java
@PutMapping("/{id}")
public Result<Void> update(@PathVariable Long id,
                           @Valid @RequestBody EmpUpdateRequest request) {
    service.update(id, request);
    return Result.success();
}
~~~

更新 DTO 只暴露允许修改的字段，不要直接把数据库实体作为所有接口的输入对象。

### 1.2 查询回显：resultMap 与嵌套结果映射

术语：查询回显指点击"编辑"时先调用 `GET /emps/{id}` 查出完整数据，前端把数据填回表单，用户改完再提交。员工详情往往包含部门名称、工作经历列表等多个部分，属于"一对一 + 一对多"的嵌套结构，需要嵌套结果映射。

先区分 `resultType` 与 `resultMap`：

| 映射方式 | 原理 | 适用场景 |
|---|---|---|
| `resultType` | 按"列名 = 属性名"自动映射（开启驼峰映射后 `dept_id` 对应 `deptId`） | 单表查询、字段能一一对上的简单结果 |
| `resultMap` | 显式声明列与属性的对应关系，支持嵌套对象、集合 | 多表关联、列名与属性名不一致、一对多/一对一嵌套 |

一句话概括：字段能自动对上就用 `resultType`，出现多表嵌套或名字对不上时用 `resultMap`。

`<association>` 映射"一对一"（员工所属部门），`<collection>` 映射"一对多"（员工的多条工作经历）：

~~~xml
<resultMap id="empResultMap" type="com.example.pojo.Emp">
    <id column="id" property="id"/>
    <result column="username" property="username"/>
    <result column="name" property="name"/>
    <result column="gender" property="gender"/>
    <result column="job" property="job"/>
    <result column="dept_id" property="deptId"/>

    <!-- 一对一：部门列封装成嵌套对象 -->
    <association property="dept" javaType="com.example.pojo.Dept">
        <id column="d_id" property="id"/>
        <result column="d_name" property="name"/>
    </association>

    <!-- 一对多：工作经历列封装成集合 -->
    <collection property="exprList" ofType="com.example.pojo.EmpExpr">
        <id column="e_id" property="id"/>
        <result column="begin" property="begin"/>
        <result column="end" property="end"/>
        <result column="company" property="company"/>
        <result column="e_job" property="job"/>
    </collection>
</resultMap>

<select id="getById" resultMap="empResultMap">
    select e.id, e.username, e.name, e.gender, e.job, e.dept_id,
           d.id as d_id, d.name as d_name,
           x.id as e_id, x.begin, x.end, x.company, x.job as e_job
    from emp e
    left join dept d on e.dept_id = d.id
    left join emp_expr x on x.emp_id = e.id
    where e.id = #{id}
</select>
~~~

三个必须注意的细节：

- 关联查询中列名会重复（`id`、`name`、`job` 都出现多次），必须用别名区分，否则 MyBatis 无法判断该列属于哪个嵌套对象，例如上面的 `d_id`、`e_job`。
- `<id>` 用来标识唯一主键，MyBatis 依据它把同一个员工的多行结果合并为一个对象，并把多行聚合成集合。
- 一对多用 `<collection>` 时必须写 `ofType`（集合元素类型），一对一用 `<association>` 时写 `javaType`（对象类型），两者写反不会报编译错误，但映射结果为空。

如果觉得嵌套 SQL 难维护，也可以拆成两次查询（查员工 + 查工作经历），再在 Service 中组装，代价是多一次数据库往返：

~~~mermaid
flowchart LR
    A[GET /emps/id] --> B[Service 查询员工]
    B --> C{映射方式}
    C -->|resultMap 嵌套| D[一次关联查询得到 Emp + Dept + exprList]
    C -->|两次查询| E[分别查 emp 与 emp_expr 后在 Service 组装]
    D --> F[返回 JSON 供前端回显]
    E --> F
~~~

### 1.3 修改员工：`<set>` 与 `<if>` 动态更新

问题出在"部分提交"：表单可能只提交了用户改动的字段，如果 SQL 写成 `update emp set username = #{username}, name = #{name}, ...`，没有提交的字段就会被写成 `null`，把库里的原值覆盖掉。

`<set>` 标签用来包裹 `update` 的赋值部分，它会自动去掉拼接结果末尾多余的逗号；再配合 `<if>` 判断，就能做到"提交了哪个字段才更新哪个字段"。

~~~xml
<update id="update">
    update emp
    <set>
        <if test="username != null and username != ''">username = #{username},</if>
        <if test="name != null and name != ''">name = #{name},</if>
        <if test="gender != null">gender = #{gender},</if>
        <if test="job != null">job = #{job},</if>
        <if test="entrydate != null">entrydate = #{entrydate},</if>
        <if test="deptId != null">dept_id = #{deptId},</if>
        update_time = now()
    </set>
    where id = #{id}
</update>
~~~

| 注意点 | 说明 |
|---|---|
| `<set>` 自动去逗号 | 拼接后末尾多余的 `,` 会被去掉，不需要在最后一个 `<if>` 里做特殊处理 |
| `<if>` 的 `test` 写属性名 | 判断的是参数对象的属性，字符串类型要同时判 `null` 和空串 |
| 需要兜底字段 | 所有 `<if>` 都不成立时会生成空的 `set` 子句导致 SQL 语法错误，示例中保留 `update_time = now()` 作为兜底 |
| `where` 不能少 | 漏写条件会更新整张表，条件里必须有主键或明确的唯一键 |
| 区分"不修改"和"清空" | 只判 `null` 时传空串会被忽略；需要真正清空字段时要单独处理空串语义 |

`<set>` 等价于手写的 `<trim prefix="set" suffixOverrides=",">`，理解 `<trim>` 之后更容易举一反三。

Service 层还要检查影响行数：返回 0 说明 ID 不存在或已被删除，应抛出业务异常，而不是静默返回成功。

### 1.4 批量删除：`<foreach>` 处理 IN 集合

前端多选后一次性提交多个 ID，SQL 形如 `delete from emp where id in (1, 2, 3)`。集合长度不固定，必须用 `<foreach>` 动态拼接。

~~~xml
<delete id="deleteByIds">
    delete from emp where id in
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</delete>
~~~

| 属性 | 取值 | 作用 |
|---|---|---|
| `collection` | `ids` | 要遍历的集合名，与 Mapper 方法参数名一致 |
| `item` | `id` | 每次迭代的元素名，`#{id}` 才能取到值 |
| `open` / `close` | `(` / `)` | 在拼接结果前后加括号，形成 `in (...)` |
| `separator` | `,` | 元素之间用逗号分隔 |

Mapper 接口与 Controller：

~~~java
void deleteByIds(@Param("ids") List<Integer> ids);
~~~

~~~java
@DeleteMapping
public Result<Void> delete(@RequestParam List<Integer> ids) {
    empService.deleteByIds(ids);
    return Result.success();
}
~~~

注意点：

- `collection` 使用了自定义名字，方法参数必须加 `@Param("ids")`；只有一个 List 参数且不命名时 MyBatis 也能识别 `collection="list"`，但显式命名更清晰，也不受参数顺序影响。
- 前端用 `?ids=1,2,3` 形式传参时用 `@RequestParam List<Integer> ids`；如果用 JSON 数组提交，则改成 `@RequestBody List<Integer> ids`，两者不能混用。
- 空集合会拼出 `where id in ()` 这种非法 SQL，Service 层要先判断 `ids == null || ids.isEmpty()` 并直接返回。
- 删除员工前要处理工作经历（`emp_expr`）等关联数据，通常是先删子表再删主表，并放在同一个事务里，避免留下孤儿记录。
- `in` 集合过大时执行计划会变差，接口层应限制单次删除的数量上限。

## 2. 全局异常处理

局部 `try/catch` 会让每个 Controller 重复处理错误。全局异常处理器可以集中记录日志、转换响应和隐藏内部实现细节。

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public Result<Void> handle(Exception ex) {
        log.error("系统异常", ex);
        return Result.error("操作失败，请稍后重试");
    }
}
~~~

详细堆栈写入日志，响应只返回稳定错误码和用户可理解的消息，不要暴露 SQL、路径和密码。

参数校验异常返回 400，认证失败返回 401，权限不足返回 403，资源不存在返回 404，未预期异常记录日志并返回 500。不要对外暴露堆栈。

建议按异常类型处理：

| 异常类型 | HTTP 状态码 | 前端可见信息 |
|---|---:|---|
| 参数校验异常 | 400 | 哪个字段不合法 |
| 未登录异常 | 401 | 请先登录 |
| 权限异常 | 403 | 没有访问权限 |
| 数据不存在 | 404 | 资源不存在 |
| 业务冲突 | 409 | 例如名称已存在 |
| 未预期异常 | 500 | 操作失败，请稍后重试 |

日志中可以保存异常堆栈、请求路径和 traceId，但响应中不要返回 SQL、文件路径、堆栈和密码。

### 2.1 异常从 Mapper 抛到全局处理器的链路

异常处理的默认规则是"谁都不处理就一路向上抛"。MyBatis 遇到 SQL 错误会把 `SQLException` 包装成 `PersistenceException`（Spring 集成后常转换为 `DataAccessException`）抛出；Service 不捕获，异常继续上抛并触发事务回滚；Controller 不捕获，Spring MVC 就把它交给 `@RestControllerAdvice` 标注的全局异常处理器。

~~~mermaid
sequenceDiagram
    participant M as Mapper
    participant S as Service
    participant C as Controller
    participant A as GlobalExceptionHandler
    participant F as 前端
    M->>S: 抛 SQLException / DataAccessException
    S->>C: 不捕获，异常上抛并回滚事务
    C->>A: 不捕获，交给 Spring MVC
    A->>A: 按 @ExceptionHandler 匹配异常类型
    A->>F: 返回统一响应 Result(code, msg, data)
~~~

~~~mermaid
flowchart TD
    A[Mapper 抛异常] --> B{Service 是否捕获}
    B -->|捕获后吞掉| C[调用方以为成功，事务不回滚，问题被隐藏]
    B -->|不捕获| D[事务回滚，异常继续上抛]
    D --> E{Controller 是否捕获}
    E -->|不捕获| F["@RestControllerAdvice 全局处理"]
    E -->|捕获后自行处理| G[局部处理，容易各写一套响应格式]
    F --> H[记录日志 + 返回统一响应]
~~~

为什么不要在 Service 里吞异常：

| 写法 | 后果 |
|---|---|
| `catch (Exception e) { e.printStackTrace(); }` 后继续返回 | 调用方以为成功；异常没有抛出，`@Transactional` 不会回滚，可能出现写了一半的数据 |
| catch 后返回 `null` 或 `false` | 错误原因丢失，前端只能显示"操作失败"，排查要靠翻日志猜 |
| catch 后包装成新异常再抛出 | 可行，但要把原异常作为 cause 传入，且不要反复包装造成堆栈难读 |
| 在 `@Transactional` 方法内 catch 后不抛出 | 事务不会回滚；如果内层方法已把事务标记为 rollback-only，提交时还可能抛 `UnexpectedRollbackException` |

正确做法：需要转换异常语义时重新包装并抛出（保留 cause），不需要转换就直接上抛，把"怎么展示给用户"交给全局异常处理器。确实需要"失败也不影响主流程"的场景（例如写操作日志），才在最小范围内捕获，并把异常记录清楚，这一点与第 13 章操作日志的事务取舍一致。

### 2.2 自定义业务异常与统一响应结果

术语：业务异常表示"用户操作不合法"这类可预期的错误（例如"部门下还有员工，不能删除"），要和程序 bug、数据库故障区分开。它通常继承 `RuntimeException`：运行期异常不需要在方法签名上声明，并且是 Spring 事务默认的回滚条件。

统一响应结果是所有接口共用的返回结构，一般包含三个字段：

| 字段 | 含义 | 约定 |
|---|---|---|
| `code` | 业务状态码 | 成功固定为 `1`，失败用其他值（如 `0` 或具体错误码） |
| `msg` | 提示信息 | 可以直接展示给用户 |
| `data` | 业务数据 | 失败时为 `null` |

~~~java
public class Result<T> {
    private Integer code; // 1 成功，0 失败
    private String msg;
    private T data;

    public static <T> Result<T> success() {
        return build(1, "success", null);
    }

    public static <T> Result<T> success(T data) {
        return build(1, "success", data);
    }

    public static <T> Result<T> error(String msg) {
        return build(0, msg, null);
    }

    private static <T> Result<T> build(Integer code, String msg, T data) {
        Result<T> result = new Result<>();
        result.code = code;
        result.msg = msg;
        result.data = data;
        return result;
    }

    // 省略 getter/setter
}
~~~

自定义业务异常：

~~~java
public class BizException extends RuntimeException {
    public BizException(String message) {
        super(message);
    }
}
~~~

Service 在业务规则不满足时抛出它：

~~~java
public void deleteById(Integer id) {
    long count = empMapper.countByDeptId(id);
    if (count > 0) {
        throw new BizException("部门下还有员工，不能删除");
    }
    deptMapper.deleteById(id);
}
~~~

全局异常处理器按类型分别处理，业务异常与兜底异常可以共存：

~~~java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BizException.class)
    public Result<Void> handleBiz(BizException ex) {
        log.warn("业务异常: {}", ex.getMessage());
        return Result.error(ex.getMessage()); // msg 可直接给用户看，code = 0
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleOther(Exception ex) {
        log.error("系统异常", ex); // 堆栈只进日志
        return Result.error("操作失败，请稍后重试");
    }
}
~~~

| 要点 | 说明 |
|---|---|
| 匹配顺序 | Spring 会选择最贴近的异常类型，业务异常优先，未被匹配的才落到 `Exception` 兜底 |
| code 约定 | 前端统一判断 `code` 是否为 1，便于在 axios 响应拦截器里集中处理 |
| 日志级别 | 业务异常用 `warn`，未预期异常用 `error` 并打印堆栈 |
| 不暴露细节 | 响应中不放 SQL、堆栈、文件路径；需要排查时用 traceId 关联日志 |
| 与其它章节呼应 | 第 09 章的文件上传接口、新增接口同样以 `Result` 作为返回类型，保持全局一致 |

## 3. 员工统计

### 3.1 职位统计

~~~sql
SELECT job, COUNT(*) AS total
FROM emp
GROUP BY job
ORDER BY total DESC;
~~~

`GROUP BY` 按职位分组，`COUNT(*)` 统计每组记录数量，`ORDER BY total DESC` 按统计结果从多到少排序。统计接口返回的数据通常转换为图表需要的 `name` 和 `value` 数组。

### 3.2 性别统计

~~~sql
SELECT gender, COUNT(*) AS total
FROM emp
GROUP BY gender;
~~~

后端把聚合结果转换成图表数组，前端使用 ECharts 等组件可视化。

统计接口可按日期和部门增加过滤条件；大数据量时考虑索引、缓存或异步汇总，避免每次请求扫描整张员工表。

图表接口常返回简单的统计 DTO：

~~~java
public record StatItem(String name, long value) {
}
~~~

SQL 的统计字段应使用明确别名，空结果也返回空数组而不是 `null`，这样前端可以直接遍历渲染。

### 3.3 行转列统计：CASE WHEN

`GROUP BY job` 得到的是"竖排"结果（每个职位一行）。如果前端要的是二维交叉表，比如"每个部门在各职位上分别有多少人"，就需要行转列，把其中一个分组维度变成列名，用条件聚合实现。

术语：条件聚合指在 `SUM`、`COUNT` 里嵌套 `CASE WHEN`，只对满足条件的行计数，从而在不增加分组的情况下得到多个统计列。

~~~sql
SELECT
    d.name AS dept_name,
    SUM(CASE WHEN e.job = 1 THEN 1 ELSE 0 END) AS head_teacher,
    SUM(CASE WHEN e.job = 2 THEN 1 ELSE 0 END) AS lecturer,
    SUM(CASE WHEN e.job = 3 THEN 1 ELSE 0 END) AS research_lead,
    SUM(CASE WHEN e.job = 4 THEN 1 ELSE 0 END) AS consultant,
    COUNT(*) AS total
FROM emp e
LEFT JOIN dept d ON e.dept_id = d.id
GROUP BY d.id, d.name
ORDER BY d.id;
~~~

| 写法 | 含义 |
|---|---|
| `SUM(CASE WHEN 条件 THEN 1 ELSE 0 END)` | 满足条件记 1 分，等价于对该条件计数 |
| `COUNT(CASE WHEN 条件 THEN 1 END)` | `COUNT` 会忽略 `null`，因此可以省略 `ELSE 0` |
| `GROUP BY d.id, d.name` | 除聚合列外，出现在 `SELECT` 中的非聚合列都要参与分组 |
| `LEFT JOIN` | 保证没有员工的部门也出现在结果里，计数为 0 |

对应的返回对象（需要开启驼峰映射，才能把 `dept_name` 映射到 `deptName`）：

~~~java
public record DeptJobStat(String deptName, int headTeacher, int lecturer,
                          int researchLead, int consultant, int total) {
}
~~~

再给一个按性别统计的例子。3.2 的 `GROUP BY gender` 适合画饼图；下面的行转列写法适合和部门组合成表格或柱状图：

~~~sql
SELECT
    d.name AS dept_name,
    COUNT(CASE WHEN e.gender = 1 THEN 1 END) AS male,
    COUNT(CASE WHEN e.gender = 2 THEN 1 END) AS female,
    COUNT(*) AS total
FROM emp e
LEFT JOIN dept d ON e.dept_id = d.id
GROUP BY d.id, d.name
ORDER BY d.id;
~~~

两种统计方式的选择：

| 需求 | 推荐写法 | 原因 |
|---|---|---|
| 每个职位有多少人，用于饼图 | `GROUP BY job` | 结果行数随分类变化，前端直接转 `name`/`value` |
| 每个部门在各职位上的人数，用于表格 | `CASE WHEN` 行转列 | 列固定，前端表格结构稳定 |
| 分类很多（如按城市统计） | 仍用 `GROUP BY` | 行转列会产生大量列，SQL 和 DTO 都难维护 |

行转列的列名是写死在 SQL 中的，分类值发生增减时要同步改 SQL 和 DTO，这是它的主要缺点。

### 3.4 用 @MapKey 接收嵌套 Map

有些报表字段不固定，或者只是临时接口，不想为每种统计都定义 DTO，这时可以用 `Map` 直接接收结果。

术语：`@MapKey` 标注在返回 `Map` 的查询方法上，用来指定结果中的哪一列作为外层 `Map` 的键；内层 `Map` 就是"列名 → 值"的一行数据。因此返回类型是 `Map<String, Map<String, Object>>`。

~~~java
@MapKey("dept_name")
Map<String, Map<String, Object>> countGroupByDept();
~~~

~~~xml
<select id="countGroupByDept" resultType="map">
    SELECT
        d.name AS dept_name,
        COUNT(CASE WHEN e.job = 1 THEN 1 END) AS head_teacher,
        COUNT(CASE WHEN e.job = 2 THEN 1 END) AS lecturer,
        COUNT(*) AS total
    FROM emp e
    LEFT JOIN dept d ON e.dept_id = d.id
    GROUP BY d.id, d.name
    ORDER BY d.id;
</select>
~~~

使用 `resultType="map"` 时，内层 Map 的键就是查询结果的列别名，所以 SQL 中的别名直接决定了返回的 JSON 字段名。上面的查询返回的结构如下（外层键取自 `@MapKey("dept_name")` 指定的列）：

~~~text
{
  "教研部": { "dept_name": "教研部", "head_teacher": 1, "lecturer": 2, "total": 3 },
  "学工部": { "dept_name": "学工部", "head_teacher": 0, "lecturer": 1, "total": 1 }
}
~~~

| 要点 | 说明 |
|---|---|
| 外层键的取值 | 由 `@MapKey` 指定，可以写列别名；取值重复时后出现的行会覆盖先出现的行 |
| 内层键的取值 | 使用 `resultType="map"` 时是列别名；中文别名会让 JSON 字段也是中文 |
| 类型安全 | 值都是 `Object`，取出后需要强转或 `Number` 转换，字段改名不会报编译错误 |
| 与 DTO 的取舍 | 报表字段固定、要长期维护时优先用 `record`/DTO；字段不固定的临时报表才用 Map |
| 返回 null | 没有数据时返回空 Map（或 null，取决于版本与配置），前端遍历前要判空 |
| 顺序 | Map 默认不保证顺序，需要固定顺序时在 SQL 中排序后由前端处理，或改用 `List` 返回 |

同一张表还可以做双向的对照：3.1 的 `GROUP BY job` 得到"职位 → 人数"，适合饼图；本节的行转列加 `@MapKey` 得到"部门 → 各职位人数"，适合表格。选择哪种取决于前端组件需要的数据形状，而不是哪种写法更"高级"。

## 4. 日志、认证和请求拦截

### 4.1 Session 与 JWT

| 方案 | 状态位置 | 优点 | 注意点 |
|---|---|---|---|
| Session | 服务端 | 易撤销、成熟 | 集群需共享会话 |
| JWT | 客户端令牌 | 无状态，适合分离架构 | 撤销和泄露处理复杂 |

JWT 的 Payload 不是加密内容，不能保存密码。签名保证完整性，不保证保密。

JWT 过期时间应较短，刷新令牌要单独管理；签名密钥放在环境变量或密钥管理系统。前端存储令牌时要评估 XSS 和 CSRF 风险。

Session 把会话数据放在服务端，浏览器只保存 Session ID；JWT 把声明和签名放在令牌中，服务端可以无状态校验。JWT 的 Payload 只是 Base64Url 编码，不是加密内容，不能保存密码、银行卡号等敏感信息。

### 4.2 Filter 与 Interceptor

~~~mermaid
flowchart LR
    A[请求] --> B[Filter]
    B --> C[Interceptor]
    C --> D[Controller]
    D --> E[响应]
~~~

Filter 属于 Servlet 规范，Interceptor 属于 Spring MVC。登录接口、静态资源等白名单要明确配置。

Filter 可以在进入 Spring MVC 前处理所有 Servlet 请求，适合字符编码、跨域和登录令牌解析；Interceptor 可以在 Controller 执行前后处理，适合登录校验、权限判断和操作日志。请求链路通常是 Filter → Interceptor → Controller → Service。

认证拦截的一般流程：

~~~mermaid
flowchart TD
    A[收到请求] --> B{是否白名单}
    B -->|是| C[继续处理]
    B -->|否| D[读取 Cookie 或 Authorization]
    D --> E{令牌有效}
    E -->|否| F[返回 401]
    E -->|是| G[写入登录用户信息]
    G --> C
~~~

白名单应包含登录、静态资源和健康检查接口，其他接口默认需要认证。权限校验不能只依赖前端按钮隐藏，后端必须再次验证。

Filter 适用于所有 Servlet 请求，Interceptor 只拦截进入 Spring MVC 的请求；静态资源、错误页和文件请求是否经过拦截要结合配置验证。认证成功后可把用户 ID 放入请求上下文，但线程池异步任务结束时要清理 ThreadLocal，避免用户信息串线。

## 5. 本章总结

- 修改由回显和提交更新组成：回显用 `resultMap` 处理一对一/一对多嵌套，更新用 `<set>` 加 `<if>` 只改提交的字段。
- 删除要考虑关联数据，批量删除用 `<foreach>` 拼 `in` 集合，并注意空集合和参数命名。
- 异常从 Mapper 一路抛到 `@RestControllerAdvice`：Service 不吞异常，业务异常承担可预期错误，统一响应以 `code`、`msg`、`data` 对外输出，成功时 `code` 为 1。
- 统计由 SQL 聚合完成，行转列用 `CASE WHEN`，字段不固定时可用 `@MapKey` 返回嵌套 Map。
- 日志和认证负责系统可维护性。
