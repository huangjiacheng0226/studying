# 四、Linux 实用操作（一）：快捷键、软件安装与服务

## 1. 终端快捷键

### 1.1 常用控制快捷键

| 快捷键 | 作用 | 场景 |
|---|---|---|
| `Ctrl+C` | 停止命令或取消输入 | 停止 `tail -f` |
| `Ctrl+D` | 退出登录或交互程序 | 退出 `su`、Python |
| `Ctrl+L` | 清屏 | 清理终端 |
| `Ctrl+A` / `Ctrl+E` | 光标到开头 / 结尾 | 编辑长命令 |
| `Ctrl+←/→` | 按单词移动 | 编辑路径 |

`Ctrl+D` 不能退出 vi/vim，应使用 `:q` 或 `:q!`。

~~~bash
tail -f /var/log/messages  # 持续查看日志
# 按 Ctrl+C 停止输出
clear                      # 与 Ctrl+L 相同
~~~

### 1.2 作业控制与行编辑快捷键

| 快捷键 | 作用 | 说明 |
|---|---|---|
| `Ctrl+Z` | 把当前任务挂起到后台 | 任务被暂停而不是结束，恢复用 `fg`，后台继续用 `bg` |
| `Ctrl+U` | 删除光标到行首的内容 | 命令前半段敲错时清掉 |
| `Ctrl+K` | 删除光标到行尾的内容 | 与 `Ctrl+U` 配合可清空整行 |
| `Ctrl+W` | 向前删除一个单词 | 以空白分词，适合删掉整段路径 |
| `Ctrl+Y` | 粘回刚被删掉的内容 | 救回 `Ctrl+U`、`Ctrl+K`、`Ctrl+W` 删掉的文本 |
| `Alt+.` | 插入上一条命令的最后一个参数 | 连按可继续往前取更早命令的参数 |
| `Tab` | 补全命令、路径、选项 | 连按两次列出全部候选 |

`Ctrl+Z` 是最容易被误当成退出的一键：它只是给前台进程发送暂停信号，被挂起的任务仍然占用内存，也仍然出现在 `ps`、`jobs` 的进程列表里。想真正结束一个任务，应在前台按 `Ctrl+C` 发送中断信号，或者对挂起、后台的任务执行 `kill`。

### 1.3 作业控制示例

~~~bash
sleep 300        # 前台运行一个长时间任务
# 按 Ctrl+Z 挂起，终端提示 [1]+  Stopped  sleep 300
jobs             # 查看当前 shell 的作业，[1] 即作业号
bg %1            # 让作业 1 转到后台继续运行
fg %1            # 把作业 1 拉回前台，再按 Ctrl+C 结束
~~~

### 1.4 history：复用历史命令

~~~bash
history                    # 查看历史命令
!grep                      # 执行最近一条以 grep 开头的命令
~~~

按 `Ctrl+R` 输入关键词反向搜索；回车执行，左右键取出后编辑。

## 2. 软件安装：yum 与 apt

`yum` 用于 CentOS 等 RPM 系统，`apt` 用于 Ubuntu 等 Debian 系统，均可自动解决依赖。

~~~bash
yum [-y] install|remove|search 软件名
apt [-y] install|remove|search 软件名
~~~

| 操作 | yum | apt |
|---|---|---|
| 安装 | `sudo yum install nginx` | `sudo apt install nginx` |
| 卸载 | `sudo yum remove nginx` | `sudo apt remove nginx` |
| 搜索 | `yum search nginx` | `apt search nginx` |

~~~bash
sudo yum -y install nmap  # 自动确认安装
sudo apt update           # 刷新软件索引
sudo apt -y install curl  # 安装 curl
~~~

### 2.1 yum、dnf 与 apt 的关系

| 工具 | 所属系统 | 说明 |
|---|---|---|
| `rpm` | RedHat 系底层 | 直接操作 `.rpm` 包，不解决依赖 |
| `yum` | CentOS 7 及更早 | 构建在 rpm 之上的包管理器，自动解决依赖 |
| `dnf` | CentOS 8 之后、Fedora | 下一代的 yum，命令基本兼容，可直接替换 yum |
| `dpkg` | Debian 系底层 | 直接操作 `.deb` 包，不解决依赖 |
| `apt` | Ubuntu、Debian | 构建在 dpkg 之上的包管理器，自动解决依赖 |

CentOS 8 及之后，`yum` 往往只是一个指向 `dnf` 的链接，执行 `yum install` 实际调用的是 dnf，因此旧教程里的 yum 命令可以照抄。

### 2.2 常用子命令对照表

| 操作 | yum / dnf | apt |
|---|---|---|
| 安装 | `sudo yum install nginx` | `sudo apt install nginx` |
| 卸载 | `sudo yum remove nginx` | `sudo apt remove nginx` |
| 刷新索引 | `sudo yum clean all` 后 `sudo yum makecache` | `sudo apt update` |
| 升级已装软件 | `sudo yum update` | `sudo apt upgrade` |
| 搜索 | `yum search nginx` | `apt search nginx` |
| 查看详情 | `yum info nginx` | `apt show nginx` |
| 查文件属于哪个包 | `yum provides /usr/bin/nmap` | `apt-file search /usr/bin/nmap` |
| 清理缓存 | `sudo yum clean all` | `sudo apt clean` |

最容易混的是 `update` 这个词：`yum update` 是升级已安装的软件，`apt update` 只是刷新索引，真正的升级命令是 `apt upgrade`。另外 `yum update` 可能连带升级内核，生产环境升级前先用 `yum list updates` 看清将要升级哪些包。

### 2.3 离线安装包：rpm 与 dpkg

| 操作 | rpm（RedHat 系） | dpkg（Debian 系） |
|---|---|---|
| 安装 | `sudo rpm -ivh 包.rpm` | `sudo dpkg -i 包.deb` |
| 卸载 | `sudo rpm -e 包名` | `sudo dpkg -r 包名` |
| 查询是否已安装 | `rpm -q 包名` | `dpkg -l 包名` |
| 查文件属于哪个包 | `rpm -qf /usr/bin/nmap` | `dpkg -S /usr/bin/nmap` |

`rpm -ivh` 的 `-i` 表示安装、`-v` 显示过程、`-h` 显示进度条。两者都只做本地安装，不会自动下载并补齐依赖，缺少依赖时直接报错退出。

要补依赖可以借用上层工具：RedHat 系用 `sudo yum localinstall 包.rpm`（新版 dnf 可用 `sudo dnf install ./包.rpm`），Debian 系用 `sudo apt install ./包.deb`。结论是能在线安装就不要手工 rpm/dpkg，把依赖解析交给 yum、dnf、apt。

### 2.4 软件源配置文件

| 系统 | 配置位置 | 要点 |
|---|---|---|
| RedHat 系 | `/etc/yum.repos.d/*.repo` | 一个文件一个源，字段含 `name`（源名称）、`baseurl`（仓库地址）、`enabled`（是否启用）、`gpgcheck`（是否校验签名） |
| RedHat 系 | `/etc/yum.conf` | 全局配置与默认值，一般无需修改 |
| Debian 系 | `/etc/apt/sources.list` | 主源列表，每行一个仓库地址 |
| Debian 系 | `/etc/apt/sources.list.d/` | 存放第三方源的附加目录 |

~~~conf
# /etc/yum.repos.d/CentOS-Base.repo
[base]
name=CentOS-$releasever - Base
baseurl=http://mirrors.example.com/centos/$releasever/os/$basearch/
enabled=1
gpgcheck=1
~~~

换成国内镜像源后要让新配置生效：RedHat 系执行 `yum clean all && yum makecache`，先清旧缓存再重建；Debian 系执行 `apt update` 刷新索引。

## 3. systemctl：管理系统服务

~~~bash
systemctl start|stop|restart|reload|status|enable|disable 服务名
~~~

| 子命令 | 作用 |
|---|---|
| `start` | 启动 |
| `stop` | 停止 |
| `status` | 查看状态 |
| `enable` | 开机自启 |
| `disable` | 取消开机自启 |

~~~bash
sudo systemctl status sshd   # 查看 SSH 服务
sudo systemctl restart sshd  # 重启服务
sudo systemctl enable sshd   # 设置开机自启
~~~

常见服务：`NetworkManager`、`network`、`firewalld`、`sshd`。

## 4. ln：创建软链接

软链接类似 Windows 快捷方式，`ls -l` 首字符为 `l`。

~~~bash
ln -s 被链接文件或目录 链接路径
ln -s /var/www/site ~/site-link  # 创建链接
rm ~/site-link                   # 只删除链接
~~~

## 5. 本篇知识点总结

1. `Ctrl+C` 停止命令，`Ctrl+D` 退出会话，`Ctrl+L` 清屏。
2. `history`、`!前缀`、`Ctrl+R` 复用历史命令。
3. CentOS 常用 `yum`，Ubuntu 常用 `apt`。
4. `systemctl` 管理服务运行和开机自启。
5. `ln -s` 创建软链接。
