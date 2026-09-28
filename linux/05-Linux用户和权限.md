# 三、Linux 用户和权限

Linux 通过“用户—用户组—权限”管理系统资源。本篇重点掌握超级管理员 root、用户切换、用户与用户组管理、权限查看和权限修改。

## 1. root 用户和用户切换

### 1.1 root 与普通用户

Linux 是多用户系统。`root` 是权限最大的超级管理员；普通用户通常只在自己的 HOME 目录中拥有完整操作权限，离开 HOME 目录后多数位置只能读取或执行，不能随意修改。

| 用户类型 | 权限特点 | 使用建议 |
|---|---|---|
| root | 几乎可以执行所有管理操作 | 仅在需要时使用，避免误删系统文件 |
| 普通用户 | 日常操作安全，权限受到限制 | 作为日常工作的默认账号 |

### 1.2 su：切换用户

`su`（switch user）用于切换到指定用户。`-` 会同时加载目标用户的环境变量和 HOME 目录，通常建议使用。

```bash
su - 用户名       # 切换到指定用户并加载其环境
su -              # 省略用户名时切换到 root
exit              # 返回上一个用户
# 也可以按 Ctrl+D 返回
```

普通用户切换到其他用户通常需要输入密码；root 切换到其他用户不需要密码。不要长期使用 root 进行普通操作。

### 1.3 sudo：临时获得管理员权限

`sudo` 只为紧随其后的一条命令临时赋予 root 权限，前提是当前用户已配置 sudo 认证。

```bash
sudo apt update             # 仅本条命令以管理员权限执行
sudo systemctl restart ssh # 管理服务时常用 sudo
```

配置 sudo 的基本步骤（需要 root）：

```bash
su -
visudo                     # 安全地打开 /etc/sudoers
```

在文件末尾加入（把 `itheima` 换成实际用户名）：

```text
itheima ALL=(ALL) NOPASSWD:ALL  # 允许该用户 sudo，且不要求输入密码
```

保存并退出后，使用 `sudo 命令` 验证配置。生产环境应按需授予权限，避免给普通账号过大的管理范围。

### 1.4 su 与 sudo 的差异

| 对比项 | su | sudo |
|---|---|---|
| 验证谁的口令 | 目标用户的口令（root 切换到别人时免密） | 当前用户自己的口令 |
| 权限范围 | 整个会话都切换为对方身份，直到 `exit` | 只授权紧随其后的一条命令 |
| 环境变量 | `su` 沿用原环境，`su -` 加载目标用户的环境 | 默认沿用当前环境，`sudo -i` 取登录环境 |
| 是否留痕 | 只记录一次登录 | 每条命令都写入 /var/log/secure（RHEL 系）或 /var/log/auth.log（Debian 系） |
| 配置依赖 | 不需要额外配置 | 依赖 /etc/sudoers 中的授权 |
| 风险 | 忘记 `exit` 会长期持有高权限 | 授权过宽等于变相给出 root |

使用建议：日常用 `sudo` 执行单条命令；需要连续做多项配置时用 `su -` 或 `sudo -i` 进入 root，办完立即 `exit`。

### 1.5 sudoers 与 visudo

| 对象 | 分工 |
|---|---|
| /etc/sudoers | 主配置文件，保存全局默认值和主要授权规则 |
| /etc/sudoers.d/ | 分片目录，把不同用户的规则拆成独立文件，便于按人维护 |
| `visudo` | 编辑主配置，保存前自动检查语法；`visudo -f 分片文件` 编辑指定分片 |
| `visudo -c` | 只检查语法，不写入文件 |

直接用 `vi /etc/sudoers` 写错语法时，sudo 会因整体解析失败而不可用，可能连修复权限的机会都没有，所以只走 visudo。

```text
alice ALL=(ALL:ALL) ALL                         # 需要输入 alice 自己的口令
%wheel ALL=(ALL:ALL) NOPASSWD: ALL              # wheel 组成员免密 sudo
bob ALL=(root) /usr/bin/systemctl restart nginx # 只允许这一条命令
```

三条规则的粒度依次收紧，按最小必要原则授权：能给单条命令就不要给 ALL，能不免密就不免密。

## 2. 用户与用户组管理

### 2.1 用户组的作用

用户可以加入多个用户组。权限既可以针对单个用户设置，也可以针对用户所属的组设置，这样便于给一批用户授予相同的访问权限。

### 2.2 创建、删除和查询用户组

以下命令通常需要 root 权限：

```bash
groupadd developers       # 创建用户组
groupdel developers       # 删除用户组（组不再被使用时执行）
getent group              # 查看系统中的所有用户组
```

`getent group` 每行包含四个字段：`组名称:密码占位符:GID:组成员列表`。占位符通常是 `x`，表示组密码存放在 `/etc/gshadow` 中；最后一个字段是逗号分隔的成员列表，列的是"以该组为附加组"的用户，把该组当主组的用户不会出现在这里。

`useradd` 的默认行为跟发行版有关：CentOS/RHEL 默认创建同名用户组并把 HOME 目录建在 `/home/用户名` 下；Debian/Ubuntu 下的 `useradd` 默认既不建同名组也不建 HOME 目录，需要显式加 `-m` 和 `-g`。所以上面第 75 行的写法在 Ubuntu 上并不会得到预期结果，跨发行版时建议把参数写全：

```bash
useradd -m -s /bin/bash -g developers bob   # 明确要求创建 HOME、指定 shell 和主组
passwd bob                                  # useradd 不会设置密码，必须再用 passwd 设置
```

### 2.3 创建、删除和查询用户

```bash
useradd -g developers -d /home/alice alice  # 指定主组和 HOME 目录
useradd bob                                # 默认创建同名组，HOME 为 /home/bob
userdel bob                                # 删除用户，保留其 HOME 目录
userdel -r alice                           # 同时删除用户的 HOME 目录
id alice                                    # 查看 alice 的 UID、GID 和所属组
id                                          # 查看当前用户信息
usermod -aG developers bob                 # 将 bob 附加到 developers 组
getent passwd                              # 查看系统中的所有用户
```

| 命令 | 用途 | 关键选项或结果 |
|---|---|---|
| `useradd` | 创建用户 | `-g` 指定已有主组；`-d` 指定 HOME |
| `userdel` | 删除用户 | `-r` 连同 HOME 一起删除 |
| `id` | 查看身份信息 | 不带用户名时查看当前用户 |
| `usermod -aG` | 增加附加组 | `-a` 表示追加，避免覆盖原有组 |
| `getent passwd` | 查询用户库 | 字段为用户名、密码占位符、UID、GID、描述、HOME、登录 shell |

`getent passwd` 每行通常有 7 个字段：`用户名:密码(x):用户ID:组ID:描述:HOME目录:登录终端`。

### 2.4 userdel 与 groupdel 的影响

| 命令 | 删掉了什么 | 没删什么 | 风险 |
|---|---|---|---|
| `userdel 用户名` | /etc/passwd、/etc/shadow、/etc/group 中的用户记录 | 家目录、crontab、运行中的进程 | 家目录变成无主目录；UID 被后建用户复用时，旧文件会被新用户接管 |
| `userdel -r 用户名` | 用户记录和 /home/用户名 | 其他位置属于该用户的文件、未清理的 crontab | 家目录中的资料一并消失，往往只能靠备份找回 |
| `groupdel 组名` | /etc/group 中的组记录 | 组成员的账号 | 仍有用户以该组为主组时会被拒绝执行，组名残留为文件的所属组 |

删除用户前先确认 4 件事：`ps -u 用户名` 是否还有进程、是否有 crontab、家目录是否有需要保留的文件、`find / -user 用户名` 是否还有文件属于该用户。

### 2.5 id：查看用户与组

`id` 把 UID、主组 GID 和全部组一次列出来，比逐个查 /etc/passwd 更直接。

```bash
id             # 当前用户：uid、gid、groups
id alice       # 查看指定用户
id -u alice    # 只看 UID
id -un alice   # 只看用户名
id -g alice    # 只看主组 GID
id -G alice    # 只看全部组 GID
groups alice   # 只看全部组名
```

```text
uid=1000(alice) gid=1000(alice) groups=1000(alice),10(wheel),1001(developers)
```

逐项解释：`uid` 是用户号，`gid` 是主组，`groups` 是主组加全部附加组。判断文件权限时要同时看主组和附加组：只要文件的所属组命中 groups 中的任意一项（包括附加组），组权限就生效。

## 3. 查看权限控制

### 3.1 使用 ls -l 查看权限

```bash
ls -l                    # 以列表形式显示权限、所有者和所属组
ls -l hello.txt          # 查看指定文件
```

典型结果中的权限字段如下：

```text
-rwxr-x---  1  alice  developers  128  Sep 6 10:00  hello.sh
```

| 部分 | 含义 |
|---|---|
| 第 1 个字符 | 文件类型：`-` 普通文件，`d` 目录，`l` 软链接 |
| 第 2～4 个字符 | 所属用户（user）的 `rwx` 权限 |
| 第 5～7 个字符 | 所属用户组（group）的 `rwx` 权限 |
| 第 8～10 个字符 | 其他用户（other）的 `rwx` 权限 |
| 后续字段 | 硬链接数、所属用户、所属组、大小、时间、名称 |

### 3.2 r、w、x 的含义

| 权限 | 普通文件 | 目录 |
|---|---|---|
| `r` 读 | 查看文件内容 | 查看目录中的文件名（如 `ls`） |
| `w` 写 | 修改文件内容 | 在目录中创建、删除、改名 |
| `x` 执行 | 将文件作为程序运行 | 进入目录或访问其内容（如 `cd`） |

例如 `drwxr-xr-x` 表示：这是目录；所属用户拥有 `rwx`，所属组拥有 `r-x`，其他用户拥有 `r-x`。`-` 表示对应权限不存在。

## 4. 修改权限和所有权

### 4.1 chmod：修改权限

只有文件所属用户或 root 可以修改权限。符号写法的格式为：

```bash
chmod [-R] 权限 文件或目录
```

```bash
chmod u=rwx,g=rx,o=x hello.txt  # u=用户，g=用户组，o=其他用户
chmod -R u=rwx,g=rx,o=x test     # -R 递归修改 test 目录及其内容
```

数字写法把 `r` 记为 4、`w` 记为 2、`x` 记为 1，再相加：

| 数字 | 权限 | 计算 |
|---:|---|---|
| 0 | `---` | 0 |
| 1 | `--x` | 1 |
| 2 | `-w-` | 2 |
| 3 | `-wx` | 2+1 |
| 4 | `r--` | 4 |
| 5 | `r-x` | 4+1 |
| 6 | `rw-` | 4+2 |
| 7 | `rwx` | 4+2+1 |

```bash
chmod 751 hello.txt  # 用户 rwx，组 r-x，其他用户 --x
```

### 4.2 chown：修改所属用户和用户组

`chown` 主要由 root 执行，用于修改文件或目录的所属用户、所属组。

```bash
chown [-R] 用户[:用户组] 文件或目录
```

```bash
chown alice hello.txt             # 修改所属用户为 alice
chown :developers hello.txt      # 只修改所属组为 developers
chown alice:developers hello.txt # 同时修改用户和用户组
chown -R alice:developers project # 递归修改目录及其内容
```

冒号 `:` 用于分隔用户和用户组；省略其中一侧即可只修改另一项。

### 4.3 chown -R 递归授权的风险

```bash
chown -R alice:alice /        # 灾难：把整个系统的所有者改成 alice
chown -R www:www /var/www     # 合理：站点目录本来就该属于 www
```

`-R` 会沿目录树一直改下去，根路径给错时影响面被成倍放大。三条实践建议：执行前先用 `ls -ld 目标` 确认路径；先对目录里的单个文件试跑一次；不要对 /、/etc、/usr、/home 使用 `-R`，改错了通常只能靠备份或快照恢复。

## 5. 默认权限与 umask

新建文件的最大默认权限是 666（默认不给执行位），新建目录是 777；实际权限等于最大权限去掉 umask 中为 1 的位。

| umask | 文件实际权限 | 目录实际权限 | 适用场景 |
|---|---|---|---|
| 022 | 644 `rw-r--r--` | 755 `rwxr-xr-x` | 系统默认值，其他用户只能读 |
| 002 | 664 `rw-rw-r--` | 775 `rwxrwxr-x` | 同组协作，组成员可以改写 |
| 077 | 600 `rw-------` | 700 `rwx------` | 私密数据，只允许自己访问 |

```bash
umask        # 查看当前掩码，例如 0022
umask 022    # 临时设置掩码，只对当前会话生效
umask -S     # 用符号形式显示，例如 u=rwx,g=rx,o=rx
```

注意 umask 是掩码而不是权限值，数值越大权限反而越小；它只对当前会话生效，要永久生效就写进 /etc/profile（全局）或 ~/.bashrc（当前用户）。

## 6. 特殊权限：SUID、SGID、Sticky Bit

| 特殊权限 | 数字 | 占用的位置 | 含义与典型例子 |
|---|---|---|---|
| SUID | 4000 | 所有者位 | 执行时临时以文件所有者的身份运行，典型例子是 /usr/bin/passwd |
| SGID | 2000 | 所属组位 | 执行时以文件所属组的身份运行；作用在目录上时，目录中新建的文件继承该目录的组，典型例子是共享目录 |
| Sticky Bit | 1000 | 其他用户位 | 只作用于目录：目录里的文件只有属主、目录属主或 root 能删，典型例子是 /tmp |

| `ls -l` 中的显示 | 含义 |
|---|---|
| `s` | 原位置本来就有 `x`，特殊位同时生效 |
| `S` | 原位置本来没有 `x`，特殊位名义存在但实际不生效 |
| `t` | 其他用户位本来就有 `x`，Sticky Bit 生效 |
| `T` | 其他用户位本来没有 `x`，Sticky Bit 不生效 |

`-rwsr-xr-x` 的 /usr/bin/passwd 带 SUID，普通用户执行它时会以 root 身份运行，所以才能改写只有 root 能写的 /etc/shadow；`drwxrwxrwt` 的 /tmp 带 Sticky Bit，其他用户虽然能在目录里创建文件，却删不掉不属于自己的文件。

```bash
chmod u+s /usr/bin/program   # 符号写法：加 SUID
chmod 4755 /usr/bin/program  # 数字写法：4 表示 SUID
chmod g+s /data/share        # 加 SGID，目录中新建文件继承目录的组
chmod +t /data/share         # 加 Sticky Bit
```

安全提醒：SUID 程序是提权攻击的常见目标，能不加就不加，加之前先确认程序是否真的需要。用 `find / -perm -4000 -type f 2>/dev/null` 可以列出系统中全部 SUID 文件，发现不认识的程序要及时排查。

## 7. 本篇知识点总结

1. `root` 是 Linux 超级管理员，普通操作应优先使用普通用户。
2. `su - 用户名` 用于切换用户，`exit` 或 `Ctrl+D` 返回上一个用户。
3. `sudo` 只对单条命令临时授权，使用前必须配置 sudoers。
4. 用户可以加入多个用户组，用户组便于批量管理权限。
5. `useradd`、`userdel`、`id`、`usermod`、`getent` 覆盖用户的创建、删除、查询和分组管理。
6. `ls -l` 的权限字段由文件类型以及 user、group、other 三组 `rwx` 组成。
7. 文件和目录中的 `r`、`w`、`x` 含义不同，目录权限尤其要结合“查看、修改、进入”理解。
8. `chmod` 修改权限，支持符号写法和 0～7 数字写法；`-R` 会递归影响目录内容。
9. `chown` 修改所属用户和用户组，通常需要 root 权限。
10. `su` 切换整个会话的身份并验证目标用户口令，`sudo` 只授权一条命令并验证自己的口令，且每条命令都会留痕。
11. sudo 的授权写在 /etc/sudoers 及其分片目录 /etc/sudoers.d/ 中，编辑只用 `visudo`，按最小必要原则授权。
12. 删除用户会留下无主家目录和游离文件，`userdel -r` 会连家目录一起删；删除前先查进程、crontab、家目录文件和该用户的文件。
13. `id` 同时显示 uid、gid 主组和 groups 附加组，判断文件权限时主组与附加组都要算。
14. 新建文件的实际权限由最大默认权限（文件 666、目录 777）去掉 umask 位得到，umask 只对当前会话生效。
15. SUID、SGID、Sticky Bit 分别占用所有者位、所属组位和其他用户位，`ls -l` 中小写 `s`/`t` 表示特殊位生效，大写 `S`/`T` 表示不生效。
