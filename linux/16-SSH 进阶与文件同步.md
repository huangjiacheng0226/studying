# 八、SSH 进阶与文件同步

> 第 02 篇学会了用 SSH 客户端连上虚拟机，本篇讲清 SSH 的握手原理、密钥免密登录、sshd 加固与文件同步。

## 1. 学习目标与前置知识

读完本篇应能：

1. 说清 SSH 握手过程，知道客户端与服务端各自持有什么密钥、会话密钥从哪里来。
2. 区分口令认证、公钥认证、键盘交互认证，说明生产环境为什么推荐公钥认证并关闭口令登录。
3. 用 `ssh-keygen` 生成密钥对，用 `ssh-copy-id` 或手工追加安装公钥，并把权限设对。
4. 用 `~/.ssh/config` 把连接参数缩成 `ssh 别名`，理解 `StrictHostKeyChecking` 的取舍。
5. 用 scp、rsync、sftp 同步文件，知道何时必须先 `--dry-run`，知道远程路径末尾斜杠的差别。
6. 按 `sshd_config` 加固清单逐项修改，并在改错之前给自己留好退路。

前置知识：[02-VMware与远程连接](02-VMware与远程连接.md) 的连接步骤、[05-Linux用户和权限](05-Linux用户和权限.md) 的 `chmod`、[06-Linux实用操作-快捷键与软件服务](06-Linux实用操作-快捷键与软件服务.md) 的 `systemctl`、[09-Linux实用操作-网络传输与端口](09-Linux实用操作-网络传输与端口.md) 的端口排查、[11-Linux实用操作-环境变量与文件传输](11-Linux实用操作-环境变量与文件传输.md) 的 scp 与 sftp 基础。第 02 篇只讲怎么连，本篇补齐原理与安全加固，不重复其操作步骤。

## 2. SSH 工作原理概述

SSH（Secure Shell）的目标是：在不安全的网络上给远程登录与文件传输套一层加密通道，让中间人既看不懂也改不了内容。它同时用到两类加密算法：

| 加密类型 | 密钥特点 | 速度 | 在 SSH 中的作用 |
| --- | --- | --- | --- |
| 非对称加密 | 公钥加密、私钥解密，两把密钥成对出现 | 慢 | 握手阶段验证身份、协商会话密钥 |
| 对称加密 | 加解密用同一把密钥 | 快 | 握手之后所有数据传输都用它 |

非对称加密运算慢，若每个数据包都用它加密，传输会慢到不可用，因此 SSH 只把它用在“握手 + 认证”这一小段，双方拿到会话密钥后全部改用对称加密。

### 2.1 客户端与服务端各持有什么

最容易混淆的是：主机密钥对和用户密钥对是两套完全不同的东西。

| 持有方 | 文件 | 是否私密 | 用途 |
| --- | --- | --- | --- |
| 服务端 | 主机私钥 `/etc/ssh/ssh_host_*_key` | 绝对私密 | 证明“我确实是这台主机” |
| 服务端 | 主机公钥 `/etc/ssh/ssh_host_*_key.pub` | 公开，会发给客户端 | 客户端用它比对已知主机指纹 |
| 客户端 | `~/.ssh/known_hosts` | 本地文件 | 记住见过的服务端公钥，防止被冒充 |
| 客户端 | 用户私钥 `~/.ssh/id_ed25519` | 绝对私密，不得外传 | 证明“我是对的那个用户” |
| 客户端 | 用户公钥 `~/.ssh/id_ed25519.pub` | 公开，可随意分发 | 手工装到服务端的 authorized_keys |
| 服务端 | `~/.ssh/authorized_keys` | 只对属主可读 | 存放允许免密登录的客户端公钥 |

### 2.2 连接建立流程

~~~mermaid
sequenceDiagram
    participant C as 客户端 ssh
    participant S as 服务端 sshd
    C->>S: 1. 建立 TCP 连接，交换版本号
    S->>C: 2. 返回服务端主机公钥与算法列表
    Note over C: 比对 known_hosts 指纹，不一致则告警并拒绝
    C->>S: 3. 交换 DH 参数，双方各自算出会话密钥
    Note over C,S: 会话密钥只在本机计算，不经过网络
    C->>S: 4. 用户认证：公钥签名或口令
    Note over S: 查 authorized_keys 校验公钥与签名
    S->>C: 5. 认证通过，进入加密会话
    C->>S: 6. 命令与文件内容走对称加密
    S->>C: 7. 执行结果与文件内容走对称加密
~~~

第 3 步常被误解成“服务端把会话密钥发给客户端”。实际用的是 Diffie-Hellman 密钥交换：双方各自生成临时参数、只交换中间结果，最终各自算出同一把会话密钥，密钥本身从不经过网络。第 4 步则是认证方式的差别所在：口令认证把密码交给服务端校验，公钥认证由客户端用私钥对挑战签名、服务端用 authorized_keys 里的公钥验签，私钥与口令都不出现在网络里。

## 3. 三种认证方式对比

| 认证方式 | 原理 | 安全性 | 适用场景 |
| --- | --- | --- | --- |
| 口令认证 password | 客户端把密码发给服务端，服务端与用户密码比对 | 低，密码可被暴力破解或撞库 | 临时登录、刚装好的机器 |
| 公钥认证 publickey | 客户端用私钥对挑战签名，服务端用 authorized_keys 里的公钥验签 | 高，私钥不出本机，还可再加口令 | 生产环境、自动化脚本 |
| 键盘交互 keyboard-interactive | 服务端通过问答逐项索取信息，后端可接 PAM 做多因素 | 中，取决于后端配置 | 动态口令、短信验证码 |

口令认证的凭据是密码本身，泄露或被撞库即可登录；公钥认证的凭据是私钥，私钥留在本地且可再加口令，服务端只存公钥；键盘交互认证不是独立凭据，只是把提问权交给服务端，背后可叠加多种验证。生产环境推荐公钥认证并关闭口令登录，理由是：

| 理由 | 说明 |
| --- | --- |
| 抵抗暴力破解 | 公钥不匹配的尝试直接失败，日志里的爆破记录大幅减少 |
| 不存在可被撞库的口令 | 凭据不是可猜的字符串，而是不可伪造的私钥签名 |
| 便于自动化与审计 | 脚本免交互；一人一密钥，回收只需删掉 authorized_keys 对应行 |
| 可再加一层保护 | 私钥设 passphrase 后配合 ssh-agent，等于“密钥 + 口令”双因素 |

前提必须牢记：关闭口令登录之前，先确认公钥已装好并能成功登录，否则会把自己锁在服务器外面。

## 4. 生成与管理密钥对

`ssh-keygen` 一次产出两个文件：私钥（无扩展名，如 `id_ed25519`）和公钥（`.pub` 结尾）。私钥自己留着，公钥发给要登录的服务器。

| 算法 | 命令 | 选择建议 |
| --- | --- | --- |
| Ed25519 | `ssh-keygen -t ed25519` | 现代首选，密钥短、签名快、安全性高 |
| RSA | `ssh-keygen -t rsa -b 4096` | 兼容性最好，位数至少 2048，老设备连不上 Ed25519 时用它 |
| ECDSA | `ssh-keygen -t ecdsa` | 用得少，仅在对方只支持 ECDSA 时使用 |

`-C` 给公钥加注释，注释附在公钥末尾，装到服务端后能看出这把钥匙属于谁：

~~~bash
ssh-keygen -t ed25519 -C "ops@laptop-2024"              # 生成 Ed25519 密钥对
ssh-keygen -t rsa -b 4096 -C "ops@laptop-2024"          # 兼容旧系统的 RSA 4096 位密钥
ssh-keygen -t ed25519 -C "ci@runner" -f ~/.ssh/id_ci    # -f 指定文件名
ssh-keygen -L -f ~/.ssh/id_ed25519.pub                  # 查看公钥指纹与注释
~~~

### 4.1 把公钥装到远程主机

~~~bash
ssh-copy-id root@${SERVER_IP}                            # 追加到默认账号的 authorized_keys
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@${SERVER_IP}   # 指定公钥文件
ssh-copy-id -p 2222 user@${SERVER_IP}                    # 端口不是 22 时用 -p
cat ~/.ssh/id_ed25519.pub | ssh user@${SERVER_IP} \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh -o PreferredAuthentications=publickey user@${SERVER_IP}   # 装完立刻验证公钥认证
~~~

`ssh-copy-id` 会自动建目录、设权限且不重复追加，优先用它；手工写法必须用 `>>` 追加而不是 `>` 覆盖，因为 `authorized_keys` 每行一把公钥、可以存放很多把，用 `>` 会把别人的公钥全部清掉。

### 4.2 权限要求：为什么私钥必须是 600

OpenSSH 设有硬性检查：权限过于宽松时宁可拒绝使用。下面这些不是建议，而是不这么做就用不了：

| 路径 | 必须权限 | 不符时的后果 |
| --- | --- | --- |
| `~/.ssh` 目录 | `700` | 报 `Bad owner or permissions on ~/.ssh`，认证失败 |
| `~/.ssh/authorized_keys` | `600` | sshd 拒绝读取，日志提示 `Authentication refused: bad ownership or modes` |
| 私钥 `~/.ssh/id_ed25519` | `600` | 客户端报 `Permissions 0644 for ... are too open`，拒绝加载私钥 |
| 公钥 `~/.ssh/id_ed25519.pub` | `644` 即可 | 公钥本来就公开，宽一点没有风险 |

私钥必须 600 的原因：它就是身份凭据，等价于一把万能密码。若权限是 644，同一台机器上的其他用户可以直接读走它并冒充你登录所有登记过它的服务器；若放在他人可写的目录里，攻击者还能悄悄替换它。

~~~bash
chmod 700 ~/.ssh                    # 目录只允许属主进入
chmod 600 ~/.ssh/authorized_keys    # 公钥清单只允许属主读写
chmod 600 ~/.ssh/id_ed25519         # 私钥只允许属主读写
chmod 644 ~/.ssh/id_ed25519.pub     # 公钥可以公开
chmod go-w ~                        # 家目录不能被组或其他用户写
~~~

## 5. 用 ~/.ssh/config 配置别名

每次都敲 `ssh -i ~/.ssh/id_ci -p 2222 ops@${SERVER_IP}` 既难记又容易写错，`~/.ssh/config` 可以为这串参数起别名，之后只写 `ssh 别名`：

~~~conf
Host node01
    HostName 192.168.88.130
    User ops
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
~~~

`HostName`、`User`、`Port`、`IdentityFile` 分别对应主机、用户、端口与私钥；`ServerAliveInterval` 每隔指定秒数发一次保活包，配合 `ServerAliveCountMax` 在连续多次无响应后断开，能解决“放着不动几分钟再敲命令就没反应”的 NAT 超时问题。

~~~bash
ssh node01                                  # 等价于 ssh -p 2222 ops@192.168.88.130
scp app.jar node01:/opt/app/                # scp 同样按别名解析主机
ssh -G node01                               # 打印最终生效的参数，排查配置覆盖
~~~

`scp`、`rsync` 读的是同一份配置，所以三种操作都能用别名。`ssh -G` 不真正连接，只输出合并后的参数，能看出端口、用户名、IdentityFile 最终取了哪个值。

### 5.1 StrictHostKeyChecking 的取舍

`known_hosts` 记录服务端主机公钥，`StrictHostKeyChecking` 决定首次连接或指纹变化时怎么办：

| 取值 | 行为 | 优点 | 风险 |
| --- | --- | --- | --- |
| `ask`（默认） | 打印指纹并询问，答 yes 才继续 | 能人工核对指纹 | 自动化脚本会卡在交互上 |
| `accept-new` | 首次自动接受并记录，指纹变化仍报错 | 自动化可用，且保留变化检测 | 首次连接无法人工核对 |
| `no` | 从不校验，一律接受并记录 | 省事，脚本永不卡住 | 完全不防中间人攻击，生产不要用 |

做法：交互登录用 `ask`，脚本用 `accept-new`。服务端重装系统后主机公钥会变，客户端报 `Host key verification failed`，确认对方确实重装过再清掉旧记录：

~~~bash
ssh-keygen -F ${SERVER_IP}          # 查询 known_hosts 里是否有该主机记录
ssh-keygen -R ${SERVER_IP}          # 删除该主机的旧记录
ssh-keyscan -p 2222 ${SERVER_IP}    # 手动获取主机公钥，用于预先写入
~~~

## 6. 免密登录在自动化中的用途

密钥免密登录的价值不只是少输一次密码，而是让机器之间可以自己完成登录，这是自动化的前置条件。

| 场景 | 用密钥登录之前 | 用密钥登录之后 |
| --- | --- | --- |
| scp 传文件 | 每次要人工输密码，脚本无法执行 | 免交互完成，可写进脚本 |
| rsync 定时同步 | 备份脚本得明文写密码，极不安全 | 可直接放进 crontab |
| 批量执行命令 | 逐台登录、逐台手工操作 | for 循环遍历主机列表一次执行 |

自动化要真正跑起来，还需三个条件同时成立：目标机 `authorized_keys` 里已登记你的公钥；脚本用别名或完整参数指明端口与私钥；`known_hosts` 已有该主机记录，或用 `accept-new` 避免交互卡住。

~~~bash
for HOST in node01 node02 node03; do                       # 1. 给一批机器分发公钥
    ssh-copy-id -i ~/.ssh/id_ed25519.pub "${HOST}"
done
for HOST in node01 node02 node03; do                       # 2. BatchMode 验证免密
    ssh -o BatchMode=yes "${HOST}" "hostname && uptime"
done
~~~

`-o BatchMode=yes` 禁止一切交互提问，写自动化脚本时加上它，能让问题当场暴露而不是让任务在半夜静默卡死。

### 6.1 用密钥登录多台机器的常见做法

| 做法 | 具体操作 | 优点 | 缺点与风险 |
| --- | --- | --- | --- |
| 一把私钥通用 | 同一对密钥的公钥分发给所有机器 | 配置最省事 | 一处私钥泄露，所有机器同时失守 |
| 一机一密钥 | 每台目标机用独立密钥对 | 影响面可控，便于逐台回收 | 管理成本高，本地要维护多把私钥 |
| 跳板机统一入口 | 只允许从跳板机连业务机 | 业务机不暴露公网，审计集中 | 跳板机成为关键节点，需重点加固 |
| 密钥加 agent 转发 | 私钥留本地，通过 agent 转发凭据 | 私钥不落到远程磁盘 | 转发期间获得 root 的机器可借用凭据 |

生产常见组合是“跳板机 + 一机一密钥 + 私钥设口令”：不同用途的密钥分开生成，`config` 里用不同 Host 块区分，注释写明归属，人员离职时按注释逐行回收。

## 7. sshd_config 安全加固

服务端的 SSH 行为由 `/etc/ssh/sshd_config` 控制。加固思路：减少可被尝试的入口、限制谁能登录、及时清理空闲连接。

| 配置项 | 建议值 | 作用与理由 |
| --- | --- | --- |
| `Port` | 改成非 22，例如 `2222` | 22 端口是全网扫描的重灾区，换端口能过滤掉绝大多数自动化爆破 |
| `PermitRootLogin` | `no` | 禁止 root 直接登录，攻击者必须先破普通账号再提权 |
| `PasswordAuthentication` | `no` | 关闭口令认证，暴力破解密码这条路直接消失 |
| `PubkeyAuthentication` | `yes` | 显式开启公钥认证，与上一条配套 |
| `MaxAuthTries` | `3` 到 `5` | 单次连接允许的认证失败次数，超过即断开 |
| `ClientAliveInterval` | `300` | 服务端每隔多少秒向客户端发一次探测包 |
| `ClientAliveCountMax` | `2` | 连续多少次探测无响应就断开，自动清理空闲会话 |
| `AllowUsers` | `AllowUsers ops deploy` | 白名单，只有列出的用户能登录，密码正确也进不来 |

改 `Port` 还有两处配套：放行防火墙端口（firewalld 用 `firewall-cmd --add-port=2222/tcp --permanent`，ufw 用 `ufw allow 2222/tcp`）；云服务器同步改安全组并禁用 22 端口入站。RHEL 系开启 SELinux 时还要执行 `semanage port -a -t ssh_port_t -p tcp 2222`。

服务名在两系不同，写错会一直看不到效果：RHEL 系是 `sshd`，Debian 系是 `ssh`，可用 `systemctl list-units --type=service | grep ssh` 确认。另外 Ubuntu 22.04 起会 `Include` 目录 `/etc/ssh/sshd_config.d/`，那里的配置会覆盖主文件。

### 7.1 修改流程与自我保护

改 SSH 配置最大的风险是把自己锁在外面，必须遵守两条纪律：先检查语法再重载；始终保留一个已登录成功的会话不动。

~~~bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak             # 1. 先备份，改坏了能还原
sudo vim /etc/ssh/sshd_config                                     # 2. 逐项修改，不要一次改太多
sudo sshd -t                                                      # 3. 只检查语法，不重启服务
sudo sshd -T | grep -iE 'port|permitrootlogin|passwordauth'       # 4. 查看最终生效值
sudo systemctl reload sshd                                        # 5. RHEL 系：重载配置
sudo systemctl reload ssh                                         #    Debian 系：服务名是 ssh
~~~

`sshd -t` 是保命命令：只做语法解析、不触碰运行中的进程，配置写错时会指出第几行有问题，此时服务仍在用旧配置工作，还有机会改回来。改完新开一个终端窗口测试登录，原会话不要关：新窗口连不上时能在原会话里还原备份，先关了会话就没有退路。`reload` 不中断已建立的连接而 `restart` 会，改配置优先用 `reload`。

## 8. 文件同步：scp、rsync 与 sftp

三种工具都走 SSH 通道，能 SSH 登录就能传文件，区别在设计目标：

| 对比项 | scp | rsync | sftp |
| --- | --- | --- | --- |
| 核心用途 | 一次性复制文件或目录 | 增量同步，适合反复同步大目录 | 交互式浏览远程文件并传输 |
| 传输方式 | 每次全量复制 | 只传差异部分，可让两边一致 | 逐个文件上传下载 |
| 断点续传 | 不支持 | 支持，`-P` 等于 `--partial --progress` | 支持，用 `reget`、`reput` |
| 排除文件 | 不支持 | 支持 `--exclude` | 不支持 |
| 演练模式 | 无 | 有 `--dry-run`，先看会做什么 | 无 |
| 交互性 | 无，执行完即退出 | 无 | 有，可先 ls、cd 再传 |

单个文件或一次性复制用 scp；大目录反复同步、需要排除或删除多余文件时用 rsync；需要先浏览远程目录再决定传哪些用 sftp。

### 8.1 scp 与 rsync 常用写法

scp 的方向由参数顺序决定：本地路径在前是上传，远程路径在前是下载，远程路径统一写 `用户@主机:路径`。

~~~bash
scp app.jar user@${SERVER_IP}:/opt/app/                # 上传文件到远程目录
scp -r ./conf user@${SERVER_IP}:/opt/app/              # 上传整个目录必须加 -r
scp -P 2222 app.jar user@${SERVER_IP}:/opt/app/        # 大写 -P 指定端口，不是小写 -p
~~~

小写 `-p` 在 scp 里表示保留时间戳与权限，和大写 `-P`（端口）完全不同。rsync 的核心优势是增量：只传修改时间或大小发生变化的文件，第二次同步大目录会明显更快。

~~~bash
rsync -avz --progress /data/ user@${SERVER_IP}:/backup/          # -a 归档、-v 过程、-z 压缩
rsync -avz -e "ssh -p 2222" /data/ user@${SERVER_IP}:/backup/    # 端口不是 22 时指定 ssh 命令
rsync -avz --exclude='*.log' --exclude='cache/' /data/ node01:/backup/   # 排除日志与缓存
rsync -avz -P /data/big.iso user@${SERVER_IP}:/backup/           # -P 保留半截文件，可断点续传
~~~

### 8.2 远程路径末尾的斜杠

源路径末尾有没有斜杠，含义完全不同：

| 写法 | 含义 | 目标端结果 |
| --- | --- | --- |
| `/data/` | 同步 data 目录里的内容 | `/backup/` 下直接出现 data 里的文件 |
| `/data` | 同步 data 目录本身 | `/backup/` 下多出一层，变成 `/backup/data/` |

记法：末尾斜杠表示“进去”，不带斜杠表示“把它整个搬过去”。需要两边目录结构完全一致时（备份、发布）用带斜杠的写法。

### 8.3 --delete 必须先演练

`--delete` 会让目标端与源端保持一致，代价是会删掉目标端多出来的文件。一旦源路径写错（例如误写成空目录、挂载点没挂上），它会把目标端内容清空，而且不进回收站。因此铁律是：任何带 `--delete` 的命令，第一次执行前必须加 `--dry-run` 看一遍输出。

~~~bash
rsync -avz --dry-run --delete /data/ user@${SERVER_IP}:/backup/ | grep -E 'deleting'   # 只看会删哪些
~~~

输出里以 `deleting` 开头的行就是将被删除的文件，确认它们确实是应该删掉的冗余副本，再去掉 `--dry-run` 真正执行。

## 9. sftp 交互式用法

sftp 提供类似 FTP 的交互环境，登录后可以一边看远程目录一边决定传什么，参数与 scp 一致，端口同样用大写 `-P`（如 `sftp -P 2222 user@${SERVER_IP}`）。常用命令：

| 命令 | 作用 |
| --- | --- |
| `pwd` / `ls` | 查看远程当前目录 / 列出远程目录内容 |
| `cd` / `lcd` | 切换远程目录 / 切换本地目录 |
| `put` / `get` | 上传 / 下载单个文件 |
| `mput` / `mget` | 批量上传 / 批量下载 |
| `reget` / `reput` | 断点续传式下载 / 上传 |
| `rm` / `mkdir` | 删除远程文件 / 创建远程目录 |
| `exit` | 退出会话 |

~~~bash
sftp -P 2222 user@${SERVER_IP}
sftp> cd /opt/app
sftp> lcd ~/build
sftp> put app.jar
sftp> exit
~~~

`mget`、`mput` 默认会对每个文件逐个询问确认，批量操作前先用 `prompt` 关掉询问，否则会一直卡在交互上。

## 10. 常见故障排查

| 现象 | 常见原因 | 检查命令 |
| --- | --- | --- |
| 连接超时 Connection timed out | 网络不通、防火墙丢包、云安全组未放行、IP 或端口写错 | `ping ${SERVER_IP}`、`nc -vz ${SERVER_IP} 2222`、检查安全组 |
| Connection refused | 服务端没启动、端口没监听、客户端还在连被改掉的 22 | `systemctl status sshd`、`ss -lntp \| grep ssh` |
| Permission denied (publickey) | 公钥没装到正确账号、权限不对、AllowUsers 不含该用户 | `ssh -v user@${SERVER_IP}`、`ls -ld ~/.ssh`、`cat ~/.ssh/authorized_keys` |
| Host key verification failed | 服务端重装或换了主机密钥，known_hosts 里是旧记录 | `ssh-keygen -F ${SERVER_IP}`、`ssh-keygen -R ${SERVER_IP}` |
| 连接成功后立刻被断开 | 登录 shell 配置报错、家目录不可写、磁盘写满、白名单限制 | `ssh -vvv`、`journalctl -u sshd -n 50`、`df -h` |

排查时最有效的两个动作：加 `-v`（或 `-vvv`）看客户端协商到哪一步失败；到服务端看日志（CentOS 系是 `/var/log/secure`，Debian 系用 `journalctl -u ssh`），服务端日志会写清拒绝原因。

## 11. 小白易错点

| 易错点 | 后果 | 正确做法 |
| --- | --- | --- |
| 私钥权限不是 600 | 客户端拒绝加载私钥 | `chmod 600 ~/.ssh/id_ed25519` |
| 把私钥拷到服务器或发给别人 | 私钥泄露即失去所有登记过它的主机 | 只分发 `.pub`，私钥永不出本机 |
| authorized_keys 权限 644 | sshd 拒绝读取，认证失败 | `chmod 600 ~/.ssh/authorized_keys` |
| 手工追加时用 `>` 覆盖 | 别人的公钥被全部清掉 | 用 `>>` 追加 |
| ssh-copy-id 装进了 root，登录却用普通账号 | 报 Permission denied (publickey) | 用 `-i` 指明公钥并核对目标用户 |
| 把 scp 的小写 `-p` 当端口用 | 端口不生效，连不上或行为异常 | 端口必须用大写 `-P` |
| rsync 源路径末尾斜杠写错 | 目标端多一层目录或文件散落根目录 | 分清 `/data/` 与 `/data` 的语义 |
| 不演练直接跑 `--delete` | 目标端文件被清空且不可恢复 | 先加 `--dry-run` 看 deleting 列表 |
| 在 Debian 系执行 `systemctl reload sshd` | 报 Unit not found，配置一直没生效 | 服务名用 `ssh` |
| 改 sshd_config 前不备份、不留会话 | 配置写错后把自己锁在外面 | 备份 + `sshd -t` + 保留已登录会话 |
| 先关口令认证再验证公钥 | 立刻失去登录能力 | 先确认公钥认证成功再关闭口令 |
| StrictHostKeyChecking 设成 no | 完全不防中间人攻击 | 交互用 `ask`，脚本用 `accept-new` |
| 改 Port 后忘放行防火墙与安全组 | 所有人都连不上，误以为配置写错 | 同步改防火墙与云安全组 |
| 一把通用私钥分发给所有人 | 一处泄露则全部机器失守 | 一机一密钥，离职时按注释逐行回收 |

## 12. 练习清单

| 序号 | 练习内容 | 验证方式 |
| --- | --- | --- |
| 1 | 用 `ssh-keygen -t ed25519` 生成密钥对并查看指纹 | `ssh-keygen -L -f` 输出位数与注释正确 |
| 2 | 用 `ssh-copy-id` 把公钥装到 `${SERVER_IP}` | `PreferredAuthentications=publickey` 免密登录成功 |
| 3 | 把 `~/.ssh` 权限改成 755 后登录，再改回 700 | 观察 `Bad owner or permissions` 报错并恢复 |
| 4 | 在 `~/.ssh/config` 为两台主机各建一个 Host 块 | `ssh 别名` 能连，`ssh -G 别名` 参数正确 |
| 5 | 用 scp 上传一个文件、下载一个日志文件 | 远程与本地的 `ls -l` 都能看到文件 |
| 6 | 用 rsync 同步 `/data/` 到远程 `/backup/`，并排除 `*.log` | `--dry-run` 输出中不含被排除文件 |
| 7 | 对同一目录分别用 `/data/` 与 `/data` 同步一次 | 对比目标端是否多出一层 data 目录 |
| 8 | 用 `rsync --dry-run --delete` 演练后再真正执行 | `grep deleting` 确认删除范围符合预期 |
| 9 | 修改 sshd_config 的 Port、PermitRootLogin、PasswordAuthentication 并用 `sshd -t` 校验 | reload 后从新端口连接成功，旧端口失败 |

## 13. 资料对应关系

| 主题 | 篇目 |
| --- | --- |
| SSH 连接步骤与端口填写 | [02-VMware与远程连接](02-VMware与远程连接.md) |
| chmod 与家目录权限 | [05-Linux用户和权限](05-Linux用户和权限.md) |
| systemctl 管理服务 | [06-Linux实用操作-快捷键与软件服务](06-Linux实用操作-快捷键与软件服务.md) |
| 端口与连通性排查 | [09-Linux实用操作-网络传输与端口](09-Linux实用操作-网络传输与端口.md) |
| scp 与 sftp 基础用法 | [11-Linux实用操作-环境变量与文件传输](11-Linux实用操作-环境变量与文件传输.md) |
| 脚本与 crontab 定时同步 | [13-Shell脚本与定时任务](13-Shell脚本与定时任务.md) |
| 用 grep、awk 过滤排查日志 | [14-文本处理三剑客](14-文本处理三剑客.md) |

## 14. 本篇总结

1. SSH 用非对称加密完成握手与认证、用对称加密传输数据；会话密钥由双方各自算出，不经过网络。
2. 主机密钥对证明机器身份，用户密钥对证明用户身份；私钥只留在本机，公钥可以分发。
3. 生产环境推荐公钥认证并关闭口令登录：抗爆破、没有可撞库的口令、便于自动化与逐人回收。
4. `ssh-keygen` 优先用 ed25519，兼容旧系统用 `rsa -b 4096`，`-C` 注释标明归属。
5. 权限必须收紧：`~/.ssh` 是 700、authorized_keys 与私钥是 600，权限不对一律拒绝使用。
6. `~/.ssh/config` 用 Host 别名简化连接，scp 与 rsync 共用同一份配置；StrictHostKeyChecking 交互用 `ask`、脚本用 `accept-new`，不要用 `no`。
7. 免密登录是自动化前置条件，配合 `BatchMode` 让失败立刻暴露；多机场景推荐一机一密钥并按注释回收。
8. sshd_config 加固：换端口、禁 root、关口令、开公钥、限制 MaxAuthTries 与 AllowUsers；RHEL 系服务名 `sshd`，Debian 系是 `ssh`。
9. 改配置先备份、用 `sshd -t` 校验、保留已登录会话，改完用 `reload` 生效。
10. scp 适合一次性复制，rsync 适合增量同步（`--exclude`、`-P` 断点续传），sftp 适合交互式挑文件。
11. rsync 末尾斜杠决定同步“内容”还是“目录本身”，`--delete` 必须先 `--dry-run`。
12. 排查 SSH 问题用 `ssh -v` 看客户端，用服务端日志看拒绝原因。
