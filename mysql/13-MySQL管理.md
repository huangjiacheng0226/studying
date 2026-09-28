# MySQL管理

> 本章介绍系统数据库和常用客户端工具，帮助你完成查看、备份和恢复等管理任务。


## 1.系统数据库

Mysql数据库安装完成后，自带了四个数据库，具体作用如下：

|数据库|含义|
|---|---|
|mysql|存储MySQL服务器正常运行所需要的各种信息(时区、主从、用户、权限等)|
|information_schema|提供了访问数据库元数据的各种表和视图，包含数据库、表、字段类型及访问权限等|
|performance_schema|为MySQL服务器运行时状态提供了一个底层监控功能，主要用于收集数据库服务器性能参数|
|sys|包含了一系列方便 DBA和开发人员利用 performance_schema性能数据库进行性能调优和诊断的视图|

### 1.1 四个库各自有哪些关键表
|数据库|关键表或视图|主要作用|
|---|---|---|
|information_schema|TABLES、COLUMNS、STATISTICS、SCHEMATA、PROCESSLIST|数据库元数据：每个库有哪些表、每张表有哪些列和索引、字符集与排序规则；这些表是只读视图，不能直接修改|
|performance_schema|events_statements_summary_by_digest、events_waits_summary_global_by_event_name、file_summary_by_instance|运行时性能数据：语句耗时、各类等待事件、文件与锁的统计，用来判断慢在哪一步；数据按维度聚合，只保留一段时间的采样|
|mysql|user、db、tables_priv、columns_priv、proxies_priv、plugin|账号、权限、时区、插件等服务器自身配置，是权限系统的落地表，日常不要手工改这些表|
|sys|processlist、session、schema_table_statistics、statements_with_full_table_scans|把 performance_schema 的原始数据换算成人能看懂的视图，本身几乎不存数据|

`sys` 库是 `performance_schema` 的易读视图：它把分散在多张表、以事件名和数字为主的原始记录，整理成按库、按表、按语句模板汇总的结果，字段名也接近自然语言。判断方向先用 `sys`，需要底层明细再回到 `performance_schema`。

### 1.2 三个常用查询
```SQL
-- 1. 查某张表的列信息：字段名、类型、是否可空、默认值、注释
SELECT column_name, column_type, is_nullable, column_default, column_comment
FROM information_schema.columns
WHERE table_schema = 'tlias' AND table_name = 'emp' ORDER BY ordinal_position;
```

```SQL
-- 2. 查当前连接：谁在连、从哪连、正在跑什么
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command <> 'Sleep' ORDER BY time DESC;
```

```SQL
-- 3. 查表大小与行数：按数据量倒序，快速定位大表
SELECT table_name, table_rows,
       ROUND(data_length / 1024 / 1024, 2) AS data_mb,
       ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'tlias'
ORDER BY data_length DESC;
```

`table_rows` 是 InnoDB 的估算值，只适合做横向比较；要精确行数只能 `COUNT(*)`，或在 `sys.schema_table_statistics` 里看统计信息。

## 2.常用工具

### 2.1 mysql

该mysql不是指mysql服务，而是指mysql的客户端工具

```Properties
mysql [options] [database]

options选项:
-u[username]|--user=username : 指定用户名，如-uroot
-p[password]|--password[=password] : 指定用户密码，如-p
-h[host]|--host=host : 指定服务器IP或域名，如-h127.0.0.1
-P[port]|--port=port : 指定连接端口（P为大写），如-P3306
-e statement|--execute=statement : 在客户端执行SQL语句并退出（无需进入mysql系统），但前面要跟上操作的数据库名，SQL用双引号包裹，如mysql -uroot -p mysql -e "select * from user;"
```

### 2.2 mysqladmin

mysqladmin是一个执行管理操作的客户端程序，可以用它来检查服务器的配置和当前运行状态，创建并删除数据库等

```Properties
## 通过帮助文档查看选项：
mysqladmin --help

示例选项:
create dbname : 创建指定数据库，如mysqladmin -uroot -p create text
drop dbname : 删除指定数据库，如mysqladmin -uroot -p drop text
ping : 检查MySQL服务器是否正在运行，如mysqladmin -uroot -p ping
status : 查看服务器状态，如mysqladmin -uroot -p status
shutdown : 关闭MySQL服务器（高风险操作），如mysqladmin -uroot -p shutdown
```

### 2.3 mysqlbinlog

由于服务器生成的二进制文件以二进制格式保存，所以如果想要检查这些文本的文本格式，就会使用到mysqlbinlog日志管理工具（需要管理员身份）

```Properties
mysqlbinlog [options] log-files1 log-files2 ...

mysqlbinlog log-file

options选项:
-d dbname|--database=dbname : 指定数据库名称，只列出指定的数据库相关操作，如mysqlbinlog -d mysql binlog.000001
-o n|--offset=n : 忽略日志中的前n行数据，如mysqlbinlog -o 10 binlog.000001
-r filename|--result-file=filename : 将输出的文本日志格式写到目标文件filename中,filename可以用绝对路径，如
mysqlbinlog -s binlog.000001 -r "F:\login.000001"
-s|--short-form : 让输出内容更简洁，只输出必要的信息，如mysqlbinlog -s binlog.000001
--start-datetime : 读取指定开始时间之后的事件，如mysqlbinlog --start-datetime="2025-03-18 10:00:00" binlog.000001
--stop-datetime : 读取指定结束时间之前的时间，如mysqlbinlog --stop-datetime="2025-03-18 12:00:00" binlog.000001
--start-position : 指定从二进制日志的哪个位置开始读取事件,如mysqlbinlog --start-position=100 binlog.000001
--stop-position : 指定读取二进制日志时的结束位置，如mysqlbinlog --stop-position=200 binlog.000001
```

- 在使用mysqlbinlog之前，需要先定位到binlog.000001所在文件夹，否则后面的使用要用到绝对路径或相对路径

### 2.4 mysqlshow

客户端对象查找工具，用来很快地查看存在哪些数据库、数据库中的表、表中的列和或者索引

```Properties
mysqlshow [options] [db_name [table_name [col_name]]]

options选项:
--count : 显示数据库及表的统计信息（数据库、表均可以不指定）
-i : 显示指定数据库或者指定表的状态信息

示例:
## 查询每个数据库的表的数量及表中记录的数量
mysqlshow -uroot -p --count
## 查询mysql库中每个表中的字段数及行数
mysqlshow -uroot -p mysql --count
## 查询mysql库中user表的详细信息
mysqlshow -uroot -p mysql user --count
```

### 2.5 mysqldump

mysqldump客户端工具用来备份数据库或在不同数据库之间进行数据迁移。备份内容包含创建表、及插入表的SQL语句

对于主要使用 InnoDB 的业务库，逻辑备份通常使用 `--single-transaction`，让备份在一个一致性快照中读取，减少对在线写入的影响。该选项不等价于“所有存储引擎都能无锁备份”，非事务型表仍需结合业务停写或其他方案评估。

命令行密码建议只写 `-p`，让客户端交互式读取，不要把明文密码直接写在命令、脚本或 shell 历史中。更多选项见 [mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)。

```mermaid
flowchart LR
    A[确认备份范围和磁盘空间] --> B[mysqldump 导出 SQL 文件]
    B --> C[校验文件和备份日志]
    C --> D[目标库创建或选择数据库]
    D --> E[mysql 或 source 导入]
    E --> F[抽样查询并校验数据]
```

```Properties
## 目标文件filename.sql可以用绝对路径或相对路径
mysqldump [options] db_name[tables] > filename.sql                ##备份指定数据库中的部分或全部表到目标文件
mysqldump [options] --database|-B db1 [db2 db3 ...] > filename.sql                ##备份多个指定的数据库到目标文件
mysqldump [options] --all-databases|-A > filename.sql                ##备份 MySQL 服务器上的所有数据库到目标文件

options连接选项:
-u[username]|--user=username : 指定用户名
-p[password]|--password[=password] : 指定用户密码，如-p
-h[host]|--host=host : 指定服务器IP或域名，如-h127.0.0.1
-P[port]|--port=port : 指定连接端口（P为大写），如-P3306

options输出选项:
--add-drop-database : 在每个数据库创建语句前加上drop database语句
--add-drop-table : 在每个表创建语句前加上drop table语句，默认开启，不开启（--skip-add-drop-table）
-n|--no-create-db : 不包含数据库的创建语句
-t|--no-create-info : 不包含数据表的创建语句
-d|--no-data : 不包含数据
-T filename|--tab[=filename] : 自动生成两个文件，一个.sql文件，创建表结构的语句，一个.txt文件，数据文件,例如：
## 拷贝mysql数据库下的user表
mysqldump -u root -p --tab=/path/to/export/directory mysql user 或
mysqldump -u root -p -T /path/to/export/directory mysql user
```

**-T filename\|--tab[=filename]参数**：

- 生成的两个文件文件名和表名一致

- filename指文件存放路径，可以是绝对路径，也可以是相对路径，如果当前已定位到指定路径，可以不指定filename

- -T后必须跟filename，而--tab后可以不跟filename

### 2.6 mysqlimport/source

mysqlimport是客户端数据导入工具，用来导入mysqldump加-T参数后导出的文本文件

```Properties
mysqlimport [options] db_name textfile1 [textfile2 ...]

示例：mysqlimport -uroot -p text /tmp/city.txt
```

如果需要导入sql文件，可以使用mysql中的source命令

```Properties
## filename.sql可以使用路径
source filename.sql;
```

## 3.MySQL 状态与进程管理

日常运维的入口是三类查询：进程列表回答谁在跑什么，状态变量回答现在忙不忙，系统参数回答配置到底是多少。

### 3.1 进程列表与 KILL
`SHOW PROCESSLIST` 与 `information_schema.processlist` 是同一份会话数据，后者可以用 `WHERE` 过滤、排序，也能和其他表关联。字段含义如下：

|字段|含义|异常时说明什么|
|---|---|---|
|Id|连接标识，KILL 用的就是它|连接被回收后 Id 会复用，不要长期缓存|
|User|建立连接的账号|出现不该出现的账号，先查权限与网络暴露面|
|Host|客户端来源，形如 ip:port|同一 IP 连接数暴涨，多为连接未释放或密码被爆破|
|db|当前默认数据库，未选择时为 NULL|连接串没指定库，后续 SQL 容易报 No database selected|
|Command|当前动作，Sleep 表示空闲，Query 表示正在执行|长时间 Query 说明语句慢；大量 Sleep 说明空闲连接过多|
|Time|当前状态已持续的秒数|Sleep 的 Time 大是正常的，Query 的 Time 到秒级才需要关注|
|State|语句执行阶段，如 Sending data、Waiting for table metadata lock|卡在 metadata lock 通常是被未提交事务占着表锁|
|Info|正在执行的语句，Command 不是 Query 时为 NULL|不加 FULL 时该字段被截断到 100 个字符，排查长 SQL 要用 SHOW FULL PROCESSLIST|

|命令|作用|风险|
|---|---|---|
|KILL QUERY id|只终止该会话当前执行的语句，连接保留|语句中途被中断，若属于大事务，回滚期间它仍然持有锁|
|KILL id（等价 KILL CONNECTION id）|终止整个连接，未提交事务随之回滚|连接池里的连接被强杀会报错；正在执行 DDL 时强杀可能留下较长的回滚清理|

建议先 `KILL QUERY`，确认语句真的停下再决定是否 `KILL CONNECTION`。杀大事务前先看 `SHOW ENGINE INNODB STATUS` 里的回滚进度，否则会把正在回滚误判成没生效。

### 3.2 SHOW STATUS 常用指标
`SHOW STATUS` 的值分两类：瞬时值反映当前状态，累计值从实例启动开始计数，只能看变化速率。累计值要在两个时间点各取一次再相减，或直接用 `mysqladmin extended-status -r` 看差值。

|指标|类型|含义|异常时指向什么问题|
|---|---|---|---|
|Threads_connected|瞬时值|当前打开的连接数|持续接近 max_connections，说明连接池过大或连接未归还，新连接会被拒绝|
|Threads_running|瞬时值|正在执行语句的连接数|明显高于 Threads_connected 的比例，说明并发集中在少数慢语句上|
|Max_used_connections|历史峰值|启动以来同时使用过的最大连接数|一旦等于 max_connections，说明连接数曾经打满，要复核连接池配置|
|Queries|累计值|执行过的语句总数（含存储过程中的语句）|除以 Uptime 得到平均 QPS；它下降而 Threads_running 上升，说明语句变慢了|
|Slow_queries|累计值|执行时间超过 long_query_time 的语句数|持续增长说明慢查询没被治理，要结合慢查询日志定位具体语句|
|Innodb_buffer_pool_read_requests|累计值|向缓冲池发起的读请求数|命中率的分母，单独看没有意义|
|Innodb_buffer_pool_reads|累计值|缓冲池未命中、必须走物理读的次数|与 read_requests 相除得到未命中率，长期高于 5% 说明缓冲池偏小或访问缺少局部性|
|Com_select|累计值|执行 SELECT 的次数|读写比例的基本参考，也用来确认流量是否打在预期实例上|
|Com_insert|累计值|执行 INSERT 的次数|写入暴涨时，要连同事务大小与 binlog 生成量一起看落盘压力|
|Uptime|累计值|实例已运行的秒数|数值很小说明刚重启过，排查时应先确认重启原因|

### 3.3 SHOW VARIABLES 常用参数
|参数|作用|关注点|
|---|---|---|
|max_connections|允许的最大连接数|过小会报 Too many connections，过大则内存吃紧，要和连接池上限一起调|
|wait_timeout|非交互连接的空闲超时秒数|比连接池的空闲回收时间还短时，会取到已被服务端关闭的连接而报错|
|innodb_buffer_pool_size|InnoDB 缓冲池大小|最关键的性能参数，通常占可用内存的 50% 到 70%，命中率低优先调它|
|character_set_server|服务器默认字符集|与应用、客户端不一致就会乱码，导入导出前确认三端一致|
|slow_query_log|慢查询日志开关|排查期打开，长期打开要配合日志轮转，避免日志盘写满|
|log_bin|二进制日志开关|数据恢复与复制的基础，关闭即没有增量恢复能力|

慢查询日志怎么读、Binlog 怎么解析，[14-日志](14-日志.md) 已经讲过，这里只留参数入口。

### 3.4 SHOW ENGINE INNODB STATUS 看哪几段
这条命令输出很长，命令行里用 `\G` 结尾按行展开，重点只有三段：

1. TRANSACTIONS：当前活跃事务及其开始时间、回滚进度，用来判断是否存在长时间不提交的事务。
2. LOCK WAIT 与 LATEST DETECTED DEADLOCK：前者是正在等待的锁请求（谁在等、被谁挡住），后者是最近一次死锁的事务与语句。
3. BUFFER POOL AND MEMORY：缓冲池大小与读写计数，可与 `SHOW STATUS` 的缓冲池指标互相印证。

### 3.5 mysqladmin 运维用法
|用法|命令|适用场景|
|---|---|---|
|extended-status|mysqladmin -uroot -p extended-status|一次拿到全部状态变量，加 `-r 2` 按固定间隔重复输出，用来看累计值的变化速率|
|processlist|mysqladmin -uroot -p processlist|不进客户端就能看会话列表，适合写进巡检脚本|
|status|mysqladmin -uroot -p status|一行摘要（Uptime、Threads、Questions、Slow queries），监控探针取值方便|
|flush-hosts|mysqladmin -uroot -p flush-hosts|清空主机缓存，用于账号被 too many connection errors 锁住时解锁；生产执行前要确认是误锁还是被爆破|
|flush-logs|mysqladmin -uroot -p flush-logs|轮转二进制日志与慢查询日志，备份前切一个 binlog 便于定位恢复点|
|variables|mysqladmin -uroot -p variables|列出系统变量，等同于 SHOW VARIABLES，脚本里比交互式客户端好调用|

## 4.用户与权限管理

```SQL
CREATE USER 'app'@'192.168.1.%' IDENTIFIED BY '${DB_PASSWORD}';   -- 建账号，host 决定允许哪些来源登录
GRANT SELECT, INSERT, UPDATE, DELETE ON tlias.* TO 'app'@'192.168.1.%';   -- 按库最小授权
SHOW GRANTS FOR 'app'@'192.168.1.%';   -- 查看已有权限
REVOKE DELETE ON tlias.* FROM 'app'@'192.168.1.%';   -- 回收权限
DROP USER 'app'@'192.168.1.%';   -- 删除账号
```

|mysql.user 的 host 写法|含义|典型用途|
|---|---|---|
|%|任意来源都能登录，校验只比对账号与密码|临时联调；生产上等于把入口交给所有能访问到端口的主机|
|localhost|只允许本机通过 unix socket 或回环地址登录|本机脚本与本地运维工具|

远程连接要同时满足两个条件：账号的 host 允许该来源，服务器的 `bind-address` 也没有被限制为 127.0.0.1，并且防火墙放开了端口。只改账号不放监听，表现是连接超时或 connection refused。

命令行里写明文密码会留在 shell 历史与进程列表中，可以用登录路径保存凭证，之后命令只引用路径名：

```bash
mysql_config_editor set --login-path=prod --host=${SERVER_IP} --user=app --password
mysql --login-path=prod tlias
```

权限的完整速查表与角色用法见 [18-补充内容](18-补充内容.md)。

## 5.数据导出与导入的常见组合
|写法|命令|说明|
|---|---|---|
|单表|mysqldump -uroot -p tlias emp > emp.sql|导出结构与数据，只含这一张表|
|多表|mysqldump -uroot -p tlias emp dept > two.sql|表名依次跟在库名之后|
|整库|mysqldump -uroot -p --databases tlias > tlias.sql|导出内容带建库语句，恢复时不必先创建数据库|
|只导结构|mysqldump -uroot -p -d tlias emp > emp_schema.sql|-d 等价于 --no-data，用于对比表结构或初始化空库|
|只导数据|mysqldump -uroot -p -t tlias emp > emp_data.sql|-t 等价于 --no-create-info，目标表必须已经存在|

|导入方式|写法|区别|
|---|---|---|
|source|先 mysql -uroot -p tlias 进入客户端，再执行 source /path/emp.sql|在已连接的会话中逐条执行，出错位置看得清楚，适合交互排查|
|重定向|mysql -uroot -p tlias < emp.sql|新建连接一次性导入，便于写进脚本；默认遇到错误即中止|

导入大文件时注意三点：先关唯一性校验并不会更快，索引维护与排序成本照旧，多数场景反而更慢；单条语句超过 max_allowed_packet 会直接失败，要同时调大服务端与客户端两侧的值；导出端与导入端的字符集参数必须一致，否则中文变成问号或乱码。

## 6.小白易错点

1. 用 KILL 杀掉事务后会话还在等锁：KILL QUERY 只中断当前语句，事务此前加的锁要等回滚结束才释放，回滚期间它仍是锁的持有者。
2. 把 SHOW STATUS 的累计值当成当前值：Queries、Com_select、Slow_queries 都是启动以来的累计计数，要两次采样相减才是这段时间的量。
3. 以为账号给了 % 就能远程连上：host 只决定允许哪些来源登录，服务器的 bind-address 或防火墙仍可能把连接挡在外面。
4. 用 'app'@'localhost' 去连 127.0.0.1：MySQL 把 localhost 与 127.0.0.1 当成两条不同的账号记录，匹配不上就报 Access denied。
5. 导入时字符集不一致导致乱码：文件按一种字符集导出、连接按另一种解释，中文就变成问号，导入前要统一客户端与服务端字符集。
6. 导入大文件前先关唯一性校验，结果更慢：跳过的只是校验，索引维护与排序成本不变，多数场景反而拖慢导入。
7. 导入报包大小错误就反复重试：这是 max_allowed_packet 限制，要调大参数或让导出方分小批次，重试不会成功。
8. 在生产上跑 mysqladmin flush-hosts 前没确认影响：它清空主机缓存，被锁住的账号能立刻重试，正在被限制的错误重试也会重新打进来。
9. 看到 SHOW PROCESSLIST 的 Info 为空就以为没在执行语句：不加 FULL 会截断到 100 个字符，内部操作本身也没有 Info，要结合 Command 与 State 判断。
10. 直接 KILL 掉连接池里的连接：连接池不知道连接已断开，下次取出才报错，表现为偶发的连接失效与重试。
11. 用 SET GLOBAL 改完参数就当已生效：它只影响当前实例的运行时，重启即失效，要持久化得用 SET PERSIST 或写进配置文件。
12. 把 information_schema.tables 的 table_rows 当精确行数：InnoDB 存的是估算值，误差随表增大，分页与对账不能用它。
13. 备份时把密码直接写在命令行：密码会留在历史记录与进程列表里，应改用 -p 交互输入或 mysql_config_editor 的 login-path。

## 7.本章小结

1. 四个系统库里，information_schema 提供元数据，performance_schema 提供运行时性能数据，mysql 保存账号与权限，sys 是 performance_schema 的易读视图。
2. 查表结构用 information_schema.columns，查连接用 processlist，查容量用 information_schema.tables，同时记住 table_rows 只是估算值。
3. 读进程列表要把 Command、Time、State 连起来看，才能分清会话是空闲、在跑慢查询，还是卡在锁上。
4. KILL QUERY 停语句，KILL CONNECTION 停连接并回滚未提交事务，大事务回滚要持续观察，不要反复强杀。
5. SHOW STATUS 要分清瞬时值与累计值：Threads_connected、Threads_running 看当前，其余累计值要取两次相减。
6. 缓冲池命中率由 Innodb_buffer_pool_reads 与 read_requests 估算，命中率低优先调 innodb_buffer_pool_size。
7. 参数速查围绕 max_connections、wait_timeout、innodb_buffer_pool_size、字符集、slow_query_log、log_bin 六个入口展开。
8. SHOW ENGINE INNODB STATUS 重点看事务、锁等待与死锁、缓冲池三段，再配合 mysqladmin 的 extended-status、processlist、status、flush-hosts、flush-logs、variables 做日常巡检。
9. 权限按最小授权操作，远程连接要同时满足账号 host 与 bind-address，凭证用 login-path 保存；日志细节见 [14-日志](14-日志.md)，权限速查见 [18-补充内容](18-补充内容.md)。

## 小练习

使用 `mysqldump` 导出一个演示数据库，再用 `mysql` 或 `source` 导入到另一个数据库中。
