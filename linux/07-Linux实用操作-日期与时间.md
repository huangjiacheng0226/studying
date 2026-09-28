# 四、Linux 实用操作（二）：日期、时区与系统时间

## 1. 时间相关的基本概念

Linux 里的“时间”是好几样不同的东西，先把名词分清楚，后面的命令才不会用错。

| 概念 | 含义 | 关键点 |
|---|---|---|
| UTC | 协调世界时，全球统一的时间基准 | 服务器与日志内部多以它为准 |
| 本地时间 | 系统时钟按当前时区换算后显示出来的时间 | `date` 默认显示的就是它 |
| 时区 | 本地时间与 UTC 的偏移规则，如 `Asia/Shanghai` | 配置文件是 `/etc/localtime` |
| CST | 缩写歧义：中国标准时间（UTC+8）与美国中部时间（UTC-6）都写作 CST | 沟通时直接写 UTC+8，别只写 CST |
| 硬件时钟 | 主板 CMOS 上的时钟，关机后靠电池继续走时 | 用 `hwclock` 读写 |
| 系统时钟 | 内核维护的时钟，`date` 看到的就是它 | 可用 NTP 校正 |
| Unix 时间戳 | 从 1970-01-01 00:00:00 UTC 起经过的秒数 | 与时区无关，同一时刻全球同一个值 |

开机时内核先读取硬件时钟来初始化系统时钟，系统运行期间由 NTP（chrony 或 ntpd）持续校正，必要时再手工把系统时钟写回硬件时钟。所以长期关机或刚装完系统的机器上，硬件时钟与系统时钟经常不一致。

服务器时间不准的后果比想象中严重：

- 日志时间错乱，多台机器的日志无法按时间排序，排查故障时事件顺序全是错的。
- 证书校验失败，客户端时间偏差过大时会直接报证书未生效或已过期。
- cron 定时任务、分布式锁、Token 有效期等依赖时间的逻辑都可能出错。

## 2. date 命令

### 2.1 查看和格式化

`date` 查看系统时间，`%` 标记控制格式。

~~~bash
date                         # 当前时间
date "+%Y-%m-%d %H:%M:%S"     # 年-月-日 时:分:秒
date "+%s"                    # Unix 时间戳
~~~

| 标记 | 含义 |
|---|---|
| `%Y` / `%y` | 四位 / 两位年份 |
| `%m` / `%d` | 月 / 日 |
| `%H` / `%M` / `%S` | 时 / 分 / 秒 |
| `%F` | 等价于 `%Y-%m-%d`，完整日期 |
| `%T` | 等价于 `%H:%M:%S`，完整时间 |
| `%Z` | 时区缩写 |
| `%s` | Unix 时间戳 |
| `%A` / `%a` | 星期全称（Monday）/ 星期简称（Mon） |
| `%B` / `%b` | 月份英文全称（March）/ 月份简称（Mar） |
| `%j` | 一年中的第几天（001～366） |
| `%N` | 纳秒（9 位数字） |
| `%u` | 星期几，1 表示周一，7 表示周日 |

查看日历：

~~~bash
cal            # 当月日历
cal 2026       # 指定年份的日历
cal 3 2026     # 指定年月的日历
~~~

### 2.2 日期计算

~~~bash
date -d "+1 day" "+%Y-%m-%d"   # 明天
date -d "-7 days" "+%Y-%m-%d"  # 七天前
~~~

`-d` 后面既能写相对时间，也能写“指定日期再加减若干天”：

~~~bash
date -d "next Monday" "+%F %A"          # 下个周一，带星期名
date -d "1 month ago" "+%F"             # 一个月前
date -d "2026-03-01 -3 days" "+%F"      # 指定日期往前推三天
date -d "+1 hour 30 minutes" "+%T"      # 一小时三十分钟之后
date -d "@1700000000" "+%F %T"          # 时间戳转可读时间
date +%s                                # 可读时间转时间戳
~~~

脚本里记录耗时常用时间戳：开始时 `start=$(date +%s)`，结束时 `end=$(date +%s)`，两者相减就是运行了多少秒。整数做减法比解析 `%F %T` 字符串方便，也便于写进日志后再统计。

## 3. 设置时区

~~~bash
sudo rm -f /etc/localtime
sudo ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  # 设置东八区
date "+%F %T %Z"                                             # 验证
~~~

## 4. 校准系统时间

~~~bash
# 现在推荐的方式：chrony（CentOS 8+/RHEL 8+/Ubuntu 默认套件）
sudo yum -y install chrony        # 或 sudo apt install chrony
sudo systemctl start chronyd      # 启动同步服务
sudo systemctl enable chronyd     # 开机自启
chronyc sources -v                # 查看时间源与同步状态
chronyc tracking                  # 查看本地时钟偏差
sudo chronyc makestep             # 手动校时（一次性对齐）
~~~

~~~bash
# 旧方式：ntp 套件（CentOS 7 时代）
sudo yum -y install ntp           # 安装 NTP
sudo systemctl start ntpd         # 启动同步服务
sudo systemctl enable ntpd        # 开机自启
sudo ntpdate -u ntp.aliyun.com    # 手动校准
~~~

`ntpdate` 早已停止维护，`ntpd` 在 CentOS 8+/Ubuntu 上默认已被 chrony 取代，照抄旧命令可能安装失败。新装机器优先用 chrony。

### 4.1 chrony 与 ntpd 的对比

两套套件做的是同一件事，适用场景差别却不小：

| 对比项 | chrony | ntpd |
|---|---|---|
| 现状 | 新系统默认（CentOS 8+/RHEL 8+/Ubuntu） | CentOS 7 时代的默认选择 |
| 首次同步速度 | 通常几秒到几十秒就收敛 | 可能需要几分钟 |
| 对间歇性联网的适应性 | 好，适合休眠、挂起、断网后恢复的机器 | 差，长时间离线后收敛很慢 |
| 状态查询命令 | `chronyc sources -v`、`chronyc tracking` | `ntpq -p` |
| 手动校时方式 | `chronyc makestep` | `ntpdate`（已停止维护） |
| 资源占用 | 更少 | 相对更多 |

结论：虚拟机、云主机、经常挂起恢复的开发机都用 chrony，只有维护老系统时才用 ntpd。

`/etc/chrony.conf` 里真正需要关注的配置项不多：

~~~conf
# 指定上游时间服务器；iburst 让开机时连续快速发几次请求，加快首次同步
server ntp.aliyun.com iburst

# 记录本机时钟的固有偏差，下次开机可以更快校准
driftfile /var/lib/chrony/drift

# 允许前三次同步直接跳变（偏差超过 1.0 秒就立即步进），适合开机阶段
makestep 1.0 3

# 把系统时钟同步回硬件时钟，保持两者一致
rtcsync
~~~

改完配置记得重启服务：`sudo systemctl restart chronyd`。

### 4.2 timedatectl 统一管理

时区也可以用 `timedatectl` 统一管理，比手工删改 `/etc/localtime` 更稳妥：

~~~bash
timedatectl                                   # 查看当前时间、时区和 NTP 状态
timedatectl list-timezones | grep Shanghai    # 查询可用时区
sudo timedatectl set-timezone Asia/Shanghai   # 设置时区
sudo timedatectl set-ntp true                 # 开启自动同步
~~~

`timedatectl` 输出里几个字段的含义：

| 字段 | 含义 |
|---|---|
| `Local time` | 本地时间，随时区变化 |
| `Universal time` | UTC 时间，不随时区变化 |
| `RTC time` | 硬件时钟（CMOS）的时间 |
| `Time zone` | 当前时区，例如 `Asia/Shanghai (CST, +0800)` |
| `System clock synchronized` | 系统时钟是否已与时间源同步 |
| `NTP service` | NTP 客户端服务是否开启 |

## 5. cal 与 hwclock

`cal` 看日历，`hwclock` 读写硬件时钟，两者都和时间有关但用途完全不同。

~~~bash
cal            # 当月日历，高亮今天
cal 2026       # 2026 年全年日历
cal 3 2026     # 2026 年 3 月
cal -y         # 当年全年日历，等价于 cal 2026
cal -3         # 上个月、本月、下个月
~~~

`hwclock` 用来在硬件时钟与系统时钟之间搬运时间：

~~~bash
sudo hwclock --show      # 查看硬件时钟当前时间
sudo hwclock --systohc   # 系统时钟 -> 硬件时钟（写入 CMOS）
sudo hwclock --hctosys   # 硬件时钟 -> 系统时钟（读入内核）
~~~

两个方向含义不同，使用场景也不同：

| 命令 | 方向 | 典型场景 |
|---|---|---|
| `hwclock --systohc` | 系统时钟写到硬件时钟 | 系统时间已经用 NTP 校准好，希望关机后硬件时钟也正确 |
| `hwclock --hctosys` | 硬件时钟读入系统时钟 | 虚拟机恢复快照或挂起后时间错乱、主板电池没电导致开机时间乱 |

虚拟机时间错乱、主板电池没电时，可以先用 `--hctosys` 把系统时间拉回硬件时钟的值再排查。但正常做法是让 NTP（chrony）负责同步，反复手工覆盖只会让 NTP 的偏差统计越来越乱。

## 6. 本篇知识点总结

1. `date` 查看和格式化时间。
2. `date -d` 支持日期计算。
3. 中国大陆常用 `Asia/Shanghai` 时区。
4. NTP 用于校准系统时间。
5. 时间概念要分清：UTC 是全球基准，本地时间按时区换算，Unix 时间戳与时区无关，CST 缩写有歧义。
6. 硬件时钟（`hwclock`）关机后继续走时，系统时钟（`date`）由内核维护；开机时用硬件时钟初始化，运行期间由 NTP 校正。
7. 时间同步优先用 chrony，`chronyc sources -v` 和 `chronyc tracking` 看状态，ntpd 与 ntpdate 只在维护老系统时使用。
8. `timedatectl` 可以统一查看和设置时间、时区与 NTP 开关。
9. `cal` 查看日历，`hwclock` 负责硬件时钟与系统时钟之间的双向同步。
