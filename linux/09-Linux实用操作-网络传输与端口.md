# 四、Linux 实用操作（四）：网络传输与端口

本篇围绕两条主线：一是网络通不通，用 ping、curl、wget 分别解决"能不能到""服务返回了什么""文件能不能拿下来"；二是端口被谁占用，先理解端口的分段与绑定规则，再用 ss、lsof 定位具体进程。排查顺序一般是：先 ping 看主机是否可达，再用 curl 看服务是否响应，最后用 ss 看端口被哪个进程监听。

## 1. ping：检查连通性

ping 基于 ICMP 协议，向目标主机发送回显请求并等待回显应答，用来判断网络链路是否通畅。

~~~bash
ping [-c 次数] IP或主机名
ping -c 4 www.baidu.com  # 发送 4 次
~~~

不加 `-c` 会持续发送，按 `Ctrl+C` 停止。

### 1.1 常用选项

| 选项 | 含义 | 典型用法 |
|---|---|---|
| -c 次数 | 发送指定次数的包后自动退出 | ping -c 4 ${SERVER_IP} |
| -i 间隔 | 两次发送之间的时间间隔，单位为秒 | ping -i 0.5 -c 4 ${SERVER_IP} |
| -s 包大小 | 指定每次发送的数据字节数，默认 56 | ping -s 1024 -c 4 ${SERVER_IP} |
| -W 超时 | 等待每次应答的超时时间，单位为秒 | ping -W 1 -c 4 ${SERVER_IP} |
| -4 | 只使用 IPv4 | ping -4 -c 4 ${SERVER_IP} |
| -6 | 只使用 IPv6 | ping -6 -c 4 ${SERVER_IP} |

### 1.2 输出示例

~~~text
PING www.baidu.com (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66: icmp_seq=1 ttl=52 time=8.13 ms
64 bytes from 110.242.68.66: icmp_seq=2 ttl=52 time=8.42 ms
64 bytes from 110.242.68.66: icmp_seq=3 ttl=52 time=7.96 ms
64 bytes from 110.242.68.66: icmp_seq=4 ttl=52 time=8.28 ms

--- www.baidu.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 7.960/8.197/8.420/0.176 ms
~~~

- packets transmitted / received：已发送与已收到的包数，两者相等说明这几次探测都得到了应答。
- packet loss：丢包率。0% packet loss 表示发出去的每一个包都收到了应答，链路稳定、没有明显拥塞。
- rtt min/avg/max/mdev：往返时延的最小值、平均值、最大值和抖动，mdev 越大说明时延越不稳定。
- time：整轮探测消耗的总时间；ttl 表示数据包还能经过多少跳。

丢包率不为 0 时说明链路存在丢包，可以结合 traceroute 或 mtr 继续定位是哪一跳出的问题。

### 1.3 ping 不通不等于服务不可用

ping 通只证明网络层可达，ping 不通也只说明 ICMP 这条路走不通，常见原因如下。

| 现象 | 可能原因 |
|---|---|
| 完全无应答 | 主机关机、IP 配置错误、网段不可路由 |
| Request timeout | 对方主机或中间防火墙丢弃了 ICMP 请求 |
| 100% packet loss，但端口能连 | 云服务器安全组默认不响应 ICMP，服务本身正常 |

所以判断服务是否可用要在端口层面确认，例如用 curl、telnet 或 nc 直接访问目标端口，不能只看 ping 的结果。

## 2. wget：下载文件

wget 是非交互式的下载工具，定位就是"把文件拿下来"：给一个 URL，它就把文件按远程文件名保存到当前目录，适合下载安装包、系统镜像和日志文件，也适合写在脚本里定时拉取。

~~~bash
wget [-b] URL
wget https://example.com/file.zip     # 前台下载
wget -b https://example.com/file.zip  # 后台下载
tail -f wget-log                      # 查看后台进度
~~~

### 2.1 常用选项

| 选项 | 含义 | 典型用法 |
|---|---|---|
| -O 文件名 | 指定保存的文件名 | wget -O jdk.tar.gz https://example.com/jdk-17_linux-x64_bin.tar.gz |
| -b | 后台下载，进度写入 wget-log | wget -b https://example.com/file.zip |
| -c | 断点续传，从已下载的位置继续 | wget -c https://example.com/file.zip |
| --limit-rate=速率 | 限制下载速度，避免占满带宽 | wget --limit-rate=200k https://example.com/file.zip |
| -P 目录 | 保存到指定目录 | wget -P /tmp/download https://example.com/file.zip |
| -t 次数 | 失败后的重试次数，0 表示无限重试 | wget -t 3 https://example.com/file.zip |
| --no-check-certificate | 跳过 HTTPS 证书校验，仅测试用 | wget --no-check-certificate https://example.com/file.zip |

### 2.2 示例

~~~bash
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz
wget -c -P /opt/soft https://example.com/mysql-8.0.tar.gz
wget -b --limit-rate=500k https://example.com/image.iso
tail -f wget-log
~~~

默认情况下 wget 不会覆盖已存在的完整文件，遇到同名文件会追加 `.1` 之类的后缀；如果希望接着上次的进度继续下载，记得加 `-c`。`--no-check-certificate` 只应在测试环境临时使用，正式环境跳过证书校验会带来中间人攻击风险。

## 3. curl：发送 HTTP 请求

curl 是通用的 URL 传输工具，除了下载还能发送各种 HTTP 请求，并且可以把响应头、状态码等细节打印出来，因此它更适合用来排查"服务到底返回了什么"。

~~~bash
curl URL
curl -O https://example.com/file.zip  # 按远程文件名保存
curl cip.cc                           # 查询公网 IP
~~~

`wget` 偏重下载，`curl` 更适合发送请求并查看响应。

### 3.1 常用选项

| 选项 | 含义 | 典型用法 |
|---|---|---|
| -I | 只看响应头，发送 HEAD 请求 | curl -I http://${SERVER_IP}:8080 |
| -i | 输出中同时包含响应头和响应体 | curl -i http://${SERVER_IP}:8080 |
| -L | 跟随重定向，最多自动跳转到最终地址 | curl -L http://${SERVER_IP} |
| -o 文件 | 把响应保存到指定文件 | curl -o page.html http://${SERVER_IP} |
| -O | 用远程文件名保存 | curl -O http://${SERVER_IP}/app.jar |
| -X | 指定请求方法，如 POST、PUT、DELETE | curl -X POST http://${SERVER_IP}/api/users |
| -H | 添加请求头 | curl -H "Accept: application/json" http://${SERVER_IP} |
| -d | 发送请求体，默认配合 POST 使用 | curl -d '{"name":"demo"}' http://${SERVER_IP}/api/users |
| -k | 跳过证书校验，访问自签名 HTTPS 时使用 | curl -k https://${SERVER_IP} |
| -s | 静默模式，不输出进度和错误信息 | curl -s -o /dev/null -w "%{http_code}\n" http://${SERVER_IP} |
| -w | 请求结束后按格式输出额外信息 | curl -w "%{http_code} %{time_total}\n" http://${SERVER_IP} |
| -v | 打印完整的请求与响应过程，便于排查 | curl -v http://${SERVER_IP}:8080 |

### 3.2 示例

~~~bash
curl -I http://${SERVER_IP}:8080
curl -s -o /dev/null -w "%{http_code}\n" http://${SERVER_IP}:8080
curl -X POST -H "Content-Type: application/json" -d '{"name":"demo"}' http://${SERVER_IP}:8080/api/users
curl -o app.jar http://${SERVER_IP}:8080/download/app.jar
curl -L -o page.html http://${SERVER_IP}
~~~

排查服务时，`curl -I` 一次就能同时确认三件事：端口是否可达（连不上会立刻报 Connection refused 或 Connection timed out）、返回什么状态码（200 正常、404 路径不对、500 服务内部出错）、是否重定向（返回 301/302 时响应头里带有 Location）。`curl -s -o /dev/null -w "%{http_code}\n"` 把响应体丢掉、只留下状态码，适合写在监控脚本里。

## 4. ping、curl、wget 三者对比

| 对比项 | ping | curl | wget |
|---|---|---|---|
| 工作层次 | ICMP，属于网络层 | HTTP/HTTPS 等应用层 | HTTP/HTTPS 等应用层 |
| 主要用途 | 判断主机是否可达、链路是否丢包 | 发送请求并查看响应，可调试接口 | 把文件下载保存到本地 |
| 能否看到状态码 | 不能，只有应答与丢包率 | 能，配合 -I 或 -w 输出 | 不能，只关心文件内容 |
| 能否下载保存 | 不能 | 能，-o 或 -O | 能，默认就会保存 |
| 是否跟随重定向 | 不涉及重定向 | 默认不跟随，需要加 -L | 默认跟随 |
| 典型场景 | 网络不通时先 ping 一下 | 排查服务是否正常、调试接口 | 下载安装包、镜像、日志 |

一句话区分：ping 只问能不能到，curl 关心服务返回了什么，wget 专注把文件完整拿下来。

## 5. 端口基础

IP 定位计算机，端口进一步定位应用。TCP/UDP 的端口号是 16 位无符号整数，取值范围 0～65535；其中 0 号端口被保留，不会分配给具体服务，所以实际可用的是 1～65535。按用途又分为三段：

| 范围 | 类型 | 示例 |
|---:|---|---|
| 1～1023 | 公认端口 | SSH 22、HTTPS 443 |
| 1024～49151 | 注册端口 | 应用服务 |
| 49152～65535 | 动态端口 | 临时连接 |

### 5.1 三段端口的绑定规则

| 范围 | 名称 | 谁能绑定 | 常见示例 |
|---:|---|---|---|
| 0 | 保留端口 | 不使用，由内核保留 | 无 |
| 1～1023 | 系统端口（公认端口） | 通常只有 root 或被授权的进程能绑定 | 22 SSH、80 HTTP、443 HTTPS、53 DNS |
| 1024～49151 | 注册端口 | 普通用户就可以绑定，自建服务常在这里选端口 | 3306 MySQL、6379 Redis、8080 应用服务 |
| 49152～65535 | 动态或私有端口 | 由内核为客户端临时分配，不需要手动指定 | 浏览器访问网站时本地使用的源端口 |

绑定端口还有几条容易混淆的规则：

- TCP 与 UDP 的端口空间相互独立，同一个端口号可以分别在 TCP 和 UDP 上各被一个进程监听，例如 53 端口的 TCP 与 UDP 都在工作。
- 同一个 IP 的同一个端口，在同一时刻只能被一个进程监听，再启动一个进程绑定会报 "Address already in use"。
- `0.0.0.0:80` 表示监听本机所有网卡，任何能访问到本机的地址都能连上；`127.0.0.1:80` 表示只监听回环地址，外部机器访问不到。

### 5.2 常见端口速查表

| 端口 | 协议 | 服务 | 说明 |
|---:|---|---|---|
| 22 | TCP | SSH | 远程登录，替代不安全的 Telnet |
| 80 | TCP | HTTP | 明文 Web 服务 |
| 443 | TCP | HTTPS | 加密 Web 服务 |
| 3306 | TCP | MySQL | 关系型数据库默认端口 |
| 6379 | TCP | Redis | 缓存服务默认端口 |
| 8080 | TCP | 应用服务 | Tomcat、Spring Boot 常改用这个端口 |
| 53 | TCP/UDP | DNS | 域名解析，同时使用 TCP 与 UDP |

这些公认端口可以在 `/etc/services` 里查到：

~~~bash
grep -E '^(ssh|http|https|mysql|redis)\b' /etc/services
~~~

但自定义服务通常不在其中，例如自己用 Spring Boot 起的 8080 端口，`/etc/services` 里没有记录，只能从运行中的进程用 ss 或 lsof 反查。

## 6. 查看端口占用

排查端口占有两个层次：一是扫描某台机器开放了哪些端口（nmap 从外部探测），二是看本机端口被哪个进程监听（ss、lsof 从内部查看）。

~~~bash
sudo yum -y install nmap
nmap 192.168.88.130       # 扫描开放端口
sudo yum -y install net-tools
netstat -anp | grep 6000  # 筛选端口和进程
~~~

`0.0.0.0:6000` 表示绑定所有网卡。

### 6.1 ss 的常用写法

| 写法 | 含义 |
|---|---|
| ss -lntp | 只看处于监听状态的 TCP 端口，并显示进程 |
| ss -anp | 查看全部套接字，包含已经建立的连接 |
| ss -tunlp | 同时查看 TCP 与 UDP 的监听端口 |
| netstat -tunlp | 传统等价写法，需要先安装 net-tools |
| lsof -i:端口 | 查某个端口被哪个进程占用 |

~~~bash
ss -lntp
ss -tunlp
ss -anp | grep 8080
lsof -i:8080
~~~

`ss` 是 `netstat` 的现代替代品，直接从内核读取套接字信息，在连接数很多的时候速度更快，新系统上优先使用 ss。

### 6.2 输出各列的含义

| 列 | 含义 |
|---|---|
| Netid / Proto | 协议类型，常见为 tcp 或 udp |
| State | 连接状态，LISTEN 表示正在监听等待连接；ESTAB 表示已建立连接；TIME-WAIT 表示主动关闭后等待超时回收 |
| Recv-Q / Send-Q | 接收队列与发送队列中已排队但还没被应用处理的字节数 |
| Local Address:Port | 本机监听的地址与端口，0.0.0.0 表示所有网卡 |
| Peer Address:Port | 对端的地址与端口，监听状态下通常显示 *:* |
| Process | 进程信息，格式为 进程名,pid=进程号，需要 root 或 sudo 才能看到其他用户的进程 |

## 7. 本篇知识点总结

1. `ping -c` 检查指定次数的连通性。
2. `wget` 下载，`-b` 后台下载；`curl` 发送 HTTP 请求。
3. 端口定位具体应用，22 是常见 SSH 端口。
4. `nmap` 扫描端口，`netstat -anp | grep` 查看占用。
5. ping 只回答"能不能到"，`0% packet loss` 只说明链路没丢包；ping 不通不等于服务不可用，对方可能禁用 ICMP 或被防火墙拦截，要再用 curl 从端口层面确认。
6. `wget` 与 `curl` 的区别：wget 专注把文件按远程文件名完整下载，适合安装包与镜像；curl 关注服务返回了什么，`curl -I` 能同时确认端口是否可达、状态码是多少、是否重定向。
7. 端口按用途分三段：系统端口 1～1023、注册端口 1024～49151、动态端口 49152～65535，0 号端口保留不使用。
8. 同一个 IP 的同一端口同一时刻只能被一个进程监听，`0.0.0.0:80` 表示监听所有网卡，`127.0.0.1:80` 只允许本机访问。
9. 查看端口占用常用 `ss -lntp` 看监听的 TCP 端口，`ss -tunlp` 同时看 TCP 与 UDP，`lsof -i:端口` 反查进程；ss 是 netstat 的现代替代品，速度更快。
