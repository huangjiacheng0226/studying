# 四、Linux 实用操作（六）：环境变量与文件传输

## 1. 环境变量基础

环境变量是操作系统保存的 Key-Value 信息。常见变量有 HOME、USER 和 PWD。

~~~bash
env              # 查看全部环境变量
echo $HOME       # 读取 HOME
echo ${PATH}ABC  # 用大括号明确变量边界
~~~

### 1.1 常见环境变量速查

| 变量名 | 作用 | 典型值 |
|---|---|---|
| PATH | 命令搜索路径，Shell 按顺序在其中的目录里查找可执行程序 | /usr/local/bin:/usr/bin:/bin |
| HOME | 当前用户的家目录，cd ~ 与 ~/.bashrc 都依赖它 | /root 或 /home/itheima |
| USER | 当前登录用户名，脚本里常用来区分用户 | itheima |
| SHELL | 当前使用的 Shell 程序路径 | /bin/bash |
| LANG | 语言与字符编码，影响命令行输出和程序的默认编码 | zh_CN.UTF-8 |
| PWD | 当前工作目录，执行 cd 后自动更新 | /home/itheima |
| HOSTNAME | 主机名，多机脚本中用来标记日志来源 | node01 |
| PS1 | 命令行提示符的显示格式 | [\u@\h \W]$ |

### 1.2 查看环境变量的三种方式

| 方式 | 写法 | 说明 |
|---|---|---|
| 读取单个变量 | echo $变量名 | 最常用；变量后紧跟其他字符时写 ${变量名} 明确边界 |
| 列出全部环境变量 | env | 只显示已导出的环境变量 |
| 列出全部环境变量 | printenv | 与 env 用途相同，也可用 printenv 变量名 查看单个 |

set 与它们的区别：set 显示当前 Shell 的全部变量和函数，包含尚未导出的临时变量，输出量远大于 env；env 和 printenv 只显示已导出的环境变量，也就是会继承给子进程的那一部分。

~~~bash
echo $HOME         # 查看单个变量
echo ${PATH}ABC    # 用大括号明确变量边界，否则会被当作 PATHABC
env                # 列出全部环境变量
printenv HOME      # printenv 也可以查看单个变量
printenv           # 列出全部环境变量
set                # 列出所有 Shell 变量与函数，内容比 env 多
set | grep MYNAME  # 在 set 的输出里过滤目标变量
~~~

判断一个变量有没有导出，对比两种输出即可：执行 VAR=1 后它只出现在 set 里，执行 export VAR=1 后才会同时出现在 env 里。

## 2. 环境变量分类与加载顺序

环境变量按生效范围分为系统级与用户级，最终值由配置文件的加载顺序决定，后加载的配置会覆盖先加载的同名变量。

| 级别 | 配置文件 | 适用范围 | 生效时机 |
|---|---|---|---|
| 系统级 | /etc/profile | 所有用户 | 登录 shell 启动时读取一次 |
| 系统级 | /etc/profile.d/*.sh | 所有用户 | 被 /etc/profile 依次读取 |
| 系统级 | /etc/bashrc | 所有用户 | 每次打开 bash 时被用户配置调用 |
| 用户级 | ~/.bash_profile | 当前用户 | 仅登录 shell 启动时读取 |
| 用户级 | ~/.bash_login | 当前用户 | 仅登录 shell，且 ~/.bash_profile 不存在时 |
| 用户级 | ~/.profile | 当前用户 | 仅登录 shell，且前两者都不存在时 |
| 用户级 | ~/.bashrc | 当前用户 | 每次打开交互式 bash 时读取 |

登录 shell 的典型加载顺序如下，越靠后加载优先级越高。

~~~mermaid
flowchart TD
    A[登录 shell 启动] --> B[读取 /etc/profile]
    B --> C[读取 /etc/profile.d/*.sh]
    C --> D[读取 ~/.bash_profile]
    D --> E[读取 ~/.bashrc]
    E --> F[读取 /etc/bashrc]
    F --> G[得到最终环境变量]
~~~

非登录 shell（例如 ssh 主机 命令 这种远程执行命令的方式、图形界面中新开的终端）通常不读 /etc/profile 和 ~/.bash_profile，只读 ~/.bashrc。因此只写在 ~/.bash_profile 里的变量，在非登录 shell 中可能读不到，稳妥做法是把公共变量放进 ~/.bashrc。

## 3. PATH：命令搜索路径

PATH 是一个由冒号分隔的目录顺序表，Shell 执行命令时按从左到右的顺序在这些目录里查找同名可执行文件，找到第一个就停止，后面的同名程序不会再被执行。所以 PATH 既决定“能不能找到命令”，也决定“找到的是哪一个版本”。

### 3.1 查看与定位命令

| 命令 | 作用 | 特点 |
|---|---|---|
| echo $PATH | 查看当前搜索路径 | 只显示顺序表本身 |
| which 命令名 | 显示可执行文件所在路径 | 只在 PATH 中查找，找不到外部命令时无输出 |
| type 命令名 | 显示命令的类型和来源 | 能区分别名、Shell 内建命令和外部程序，信息更全 |

~~~bash
echo $PATH                            # 查看搜索路径
which ls                              # 定位 ls 的实际路径
type ls                               # 判断 ls 是别名、内建命令还是外部程序
type cd                               # cd 是 Shell 内建命令，which 通常查不到
export PATH=$PATH:/opt/app/bin        # 临时追加目录，保留原 PATH
export PATH=/opt/app/bin:$PATH        # 需要优先命中时追加到最前面
~~~

### 3.2 永久生效与风险

临时修改只对当前 Shell 及其子进程有效，关掉终端就失效，要永久生效必须写入配置文件，再重新登录或 source 一次。

写错 PATH 的后果很严重：如果误写成 `export PATH=/opt/app/bin`，原来的 /usr/bin、/bin 全部丢失，ls、cat、vi 这类命令都会提示 command not found。此时不要退出会话，用绝对路径救急：

~~~bash
/usr/bin/ls                           # 用绝对路径临时恢复操作能力
export PATH=/usr/bin:/bin:/usr/sbin:/sbin   # 手动把 PATH 改回默认值
/usr/bin/vi ~/.bashrc                 # 用绝对路径编辑配置文件，删掉错误那行
source ~/.bashrc                      # 重新加载，确认 PATH 正常
~~~

还有一个常见建议：不要把当前目录 `.` 放进 PATH。一旦把 `.` 放在 PATH 前部，进入任意目录后执行 ls、cd 这类命令时，可能优先运行目录中同名的恶意脚本，属于明显的安全隐患。

## 4. 设置环境变量

### 4.1 临时变量与导出变量

在 Shell 中赋值有两种写法，作用范围完全不同。

| 写法 | 名称 | 作用范围 | 是否出现在 env 中 |
|---|---|---|---|
| VAR=x | Shell 变量 | 只在当前 Shell 内可见 | 否 |
| export VAR=x | 环境变量 | 当前 Shell 及之后启动的所有子进程 | 是 |

~~~bash
MYNAME=example          # 只设置 Shell 变量
bash -c 'echo $MYNAME'  # 子进程读不到，输出为空
export MYNAME=example   # 导出为环境变量
bash -c 'echo $MYNAME'  # 子进程可以读到
echo $MYNAME            # 当前 Shell 同样有效
~~~

也就是说，脚本或程序要读到的变量必须 export；只在本会话临时用一下的中间变量，普通赋值就够了。

### 4.2 永久设置的三种位置

关掉终端就失效的变量没有意义，要永久生效得写进配置文件。选择哪个文件取决于“影响谁”和“什么时候生效”。

| 位置 | 影响范围 | 生效时机 | 适用场景 |
|---|---|---|---|
| /etc/profile、/etc/profile.d/*.sh | 所有用户 | 登录 shell 启动时 | 全机统一的 PATH、JAVA_HOME 等公共变量 |
| ~/.bashrc | 当前用户 | 每次打开终端 | 个人别名、提示符、只给自己用的变量 |
| ~/.bash_profile | 当前用户 | 仅登录 shell 启动时 | 登录时执行一次的初始化动作 |

~~~bash
echo 'export MYNAME=example' >> ~/.bashrc  # 追加用户配置
source ~/.bashrc                          # 立即生效
echo $MYNAME                              # 验证
~~~

### 4.3 source 与点号

source 在当前 Shell 中逐行执行配置文件，不启动新进程，所以里面 export 的变量会留在当前会话；直接执行脚本文件则会开子进程，变量随子进程结束而消失。`.` 是 source 的同义写法。

~~~bash
source ~/.bashrc    # 重新加载用户配置，立即生效
. ~/.bashrc         # 等价写法，注意点号后要有空格
~~~

### 4.4 JAVA_HOME 与 PATH 配合

Java 相关软件通常先设置 JAVA_HOME 指向 JDK 安装目录，再把 $JAVA_HOME/bin 追加进 PATH，这样 java、javac 才能全局调用。

~~~bash
export JAVA_HOME=/opt/app/jdk17              # 声明 JDK 根目录
export PATH=$JAVA_HOME/bin:$PATH             # 让 java、javac 可被找到
echo $JAVA_HOME                              # 验证变量
java -version                                # 验证命令
/opt/app/jdk17/bin/java -version             # 用绝对路径直接执行，排除 PATH 干扰
~~~

## 5. 文件上传与下载

### 5.1 三种传输方式对比

图形面板、scp 命令和 sftp 会话底层都走 SSH 协议，密码或密钥能登录 SSH，就能用这三种方式传文件。

| 方式 | 代表工具或命令 | 适合场景 | 特点 |
|---|---|---|---|
| 图形化 SFTP 面板 | SecureCRT、FinalShell、MobaXterm | 偶尔传几个文件，习惯鼠标拖拽 | 可视化浏览两侧目录，直观但依赖软件 |
| scp 命令 | scp | 一条命令传一个文件或整个目录，写脚本批量传 | 语法简洁，传完即退出，不能交互式浏览 |
| sftp 交互式会话 | sftp | 需要连续传多个文件，边看远程目录边传 | 类似 FTP 的交互环境，可先 ls、cd 再 put、get |

### 5.2 FinalShell 图形方式

FinalShell 下方文件窗口可浏览 Linux 目录：右键远程文件可下载，将本地文件拖入目标目录可上传。本地文件通常保存在软件配置的 fsdownload 文件夹。

### 5.3 rz 与 sz

~~~bash
sudo yum -y install lrzsz  # 安装工具
rz                         # 从本地上传
sz file.txt                # 下载到本地
~~~

rz、sz 需要 FinalShell、SecureCRT、XShell 等终端软件支持。

### 5.4 scp 命令

scp 在本地和远程之间复制文件，方向由参数写法决定：本地路径写在前面就是上传，远程路径写在前面就是下载。远程路径统一写成 用户@主机:路径。

~~~bash
scp /home/itheima/app.jar itheima@node01:/opt/app/       # 上传单个文件到目录
scp -r /home/itheima/project itheima@node01:/opt/app/    # 上传整个目录必须加 -r
scp itheima@node01:/var/log/app.log ./                   # 下载远程文件到当前目录
scp -r itheima@node01:/opt/app/conf ./conf               # 下载远程目录
scp -P 2222 app.jar itheima@node01:/opt/app/             # 非默认端口用大写 -P，注意不是 -p
scp -i ~/.ssh/id_rsa app.jar itheima@node01:/opt/app/    # 用私钥免密传输
scp "itheima@node01:/opt/my app/app.jar" ./              # 路径含空格时整段加引号
~~~

### 5.5 sftp 交互式会话

sftp 登录后进入交互环境，先切换目录再传文件，适合一次传多个文件。

~~~bash
sftp itheima@node01                 # 登录远程主机，默认 22 端口
sftp -P 2222 itheima@node01         # 指定端口
~~~
~~~text
sftp> pwd                # 查看远程当前目录
sftp> ls                 # 列出远程目录内容
sftp> cd /opt/app        # 切换远程目录
sftp> mkdir logs         # 在远程创建目录
sftp> put app.jar        # 上传单个文件
sftp> mput *.jar         # 批量上传
sftp> get app.log        # 下载单个文件
sftp> mget *.log         # 批量下载
sftp> rm old.jar         # 删除远程文件
sftp> exit               # 退出会话
~~~

| 子命令 | 作用 |
|---|---|
| ls | 列出远程目录内容 |
| cd | 切换远程目录 |
| pwd | 显示远程当前目录 |
| put | 上传单个文件 |
| get | 下载单个文件 |
| mput | 批量上传，支持通配符 |
| mget | 批量下载，支持通配符 |
| mkdir | 在远程创建目录 |
| rm | 删除远程文件 |
| exit | 退出 sftp 会话 |

无论用哪种方式，路径中带空格都要用引号包起来，否则会被拆成两个参数，提示找不到文件。

## 6. 本篇知识点总结

1. env 查看环境变量，$变量名 读取值。
2. PATH 决定 Shell 查找命令的目录顺序。
3. export 设置变量；配置文件用于永久保存。
4. 追加 PATH 时要保留原 PATH。
5. FinalShell 可图形化传输，rz 上传、sz 下载。
6. echo $变量名、env、printenv 用来查看变量，set 还会显示未导出的 Shell 变量和函数。
7. 系统级配置在 /etc/profile、/etc/profile.d/*.sh、/etc/bashrc，用户级配置在 ~/.bash_profile 和 ~/.bashrc。
8. 登录 shell 依次读取系统级和用户级配置，非登录 shell 通常只读 ~/.bashrc。
9. VAR=x 只影响当前 Shell，export VAR=x 才会传给子进程。
10. which 只看 PATH，type 能区分内建命令、别名和外部程序。
11. scp 传目录要加 -r，非默认端口用 -P，用私钥用 -i；路径带空格要加引号。
12. sftp 是交互式会话，用 put、get、mget、mput 完成上传下载。
