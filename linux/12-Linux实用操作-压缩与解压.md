# 四、Linux 实用操作（七）：压缩与解压

## 1. 常见格式

| 格式 | 特点与常见系统 |
|---|---|
| .tar | 主要打包归档，体积减少有限 |
| .tar.gz | tar 后使用 gzip 压缩，Linux/macOS 常用 |
| .zip | Linux、Windows、macOS 通用 |
| .7z、.rar | Windows 中常见 |

### 1.1 打包与压缩是两件事

打包（archive）是把多个文件合并成一个大文件，只做归集，不改变内容体积；压缩（compress）是用算法减小体积，多数工具只处理单个文件。所以 .tar 只打包不压缩，.gz、.bz2、.xz 只压缩单个文件，而 .tar.gz 是先用 tar 打包、再用 gzip 压缩，两个动作合起来才能“让整个目录变小”。

### 1.2 常见格式对比

| 格式 | 典型后缀 | 典型大小（相对原文件） | 常见命令 | 适用场景 |
|---|---|---|---|---|
| tar | .tar | 接近 100%，不压缩 | tar -cvf a.tar dir/ | 只归档，便于整体拷贝或之后再压缩 |
| gz | .gz | 约 30%~50% | gzip file | 单文件压缩，速度快，默认替换原文件 |
| tar.gz | .tar.gz、.tgz | 约 30%~50% | tar -zcvf a.tar.gz dir/ | Linux 上最常见，打包加压缩一步完成 |
| bz2 | .bz2 | 约 25%~45% | bzip2 file | 压缩率比 gzip 高但速度慢，老发行版常见 |
| xz | .xz | 约 20%~40% | xz file | 压缩率最高、速度最慢，适合分发大文件 |
| zip | .zip | 约 30%~50% | zip -r a.zip dir/ | 跨平台最好，Windows 和 Linux 都能直接用 |

表中的体积比例只是量级参考，真实压缩率取决于文件类型：文本、日志、源码压缩明显，jpg、mp4 这类本身已压缩的文件几乎不再变小。

## 2. tar 命令

### 2.1 选项含义

~~~bash
tar [-c -v -x -f -z -C] 参数...
~~~

| 选项 | 含义 |
|---|---|
| -c | 创建归档 |
| -v | 显示处理过程 |
| -x | 解压 |
| -f | 指定归档文件，通常放在选项末尾 |
| -z | 使用 gzip |
| -C | 指定解压目录 |
| -t | 列出归档内容，只查看不解压 |
| -j | 使用 bzip2 算法 |
| -J | 使用 xz 算法 |
| -p | 保留文件原有权限 |
| --exclude | 排除匹配的文件或目录 |

### 2.2 示例

~~~bash
tar -cvf test.tar 1.txt 2.txt 3.txt             # 创建 tar
tar -zcvf test.tar.gz 1.txt 2.txt 3.txt         # 创建 tar.gz
tar -xvf test.tar                               # 解压到当前目录
tar -xvf test.tar -C /home/itheima             # 解压到指定目录
tar -zxvf test.tar.gz -C /home/itheima         # 解压 tar.gz
~~~

### 2.3 记忆法与常用示例

记忆法：打包 czvf，解压 xzvf，查看 tzvf，f 永远放最后。其中 c、x、t 三选一决定“干什么”（创建、解压、查看），z、j、J 三选一决定“用什么算法”，v 决定“要不要显示过程”，f 必须紧贴归档文件名。

~~~bash
tar -czvf backup.tar.gz /opt/app              # 打包目录并 gzip 压缩
tar -tzvf backup.tar.gz                       # 只查看内容清单，不解压
tar -tf backup.tar.gz                         # 去掉 -v，输出更干净
tar -xzvf backup.tar.gz -C /tmp/restore       # 解压到指定目录
tar -xzvf backup.tar.gz opt/app/conf/app.yml  # 只解压其中一个文件
tar -czvf app.tar.gz app --exclude "*.log"    # 打包时排除某类文件
tar -tzf backup.tar.gz | grep conf            # 先确认路径，再决定解压哪个
~~~

-f 后面必须是文件名，位置写错会得到意想不到的结果：

| 写法 | 结果 |
|---|---|
| tar -cvzf app.tar.gz app | 正确，f 在选项簇末尾，文件名紧跟其后 |
| tar -cfzv app.tar.gz app | 危险，f 不在末尾，紧跟的字母会被当成归档文件名，最后生成一个名字奇怪的包 |
| tar -czf app | 危险，-f 后面没有成员参数，tar 会等待标准输入，命令卡住 |
| tar -cvf app.tar.gz app.txt | 正确，先压缩包名再写成员，顺序不能颠倒 |

写完命令后不妨用 ls 和 tar -tf 各确认一次，确认归档名和内容清单都对，再删源文件。

## 3. zip 与 unzip

~~~bash
zip -r project.zip project/        # 目录必须加 -r
unzip project.zip                  # 解压到当前目录
unzip -d /tmp/project project.zip  # 解压到指定目录
~~~

| 命令 | 用途 | 关键选项 |
|---|---|---|
| zip | 创建 zip 文件 | -r 递归处理目录 |
| unzip | 解压 zip 文件 | -d 指定输出目录 |

### 3.1 常用示例

~~~bash
zip -r project.zip project/                 # 打包整个目录，-r 不能省
zip -r project.zip project/ -x "*.log"      # 打包时排除日志文件
zip archive.zip a.txt b.txt                 # 打包多个文件不需要 -r
unzip project.zip                           # 解压到当前目录
unzip project.zip -d /tmp/project           # 解压到指定目录
unzip -l project.zip                        # 只查看内容清单，不解压
unzip -o project.zip                        # 覆盖同名文件且不再询问
unzip project.zip "project/conf/*"          # 只解压匹配的路径
~~~

### 3.2 注意事项

| 事项 | 说明 |
|---|---|
| 需要单独安装 | 精简系统可能没有，用 sudo yum -y install zip unzip 安装 |
| -r 不能少 | 打包目录时不加 -r 只写入目录条目本身，解压后目录是空的 |
| 权限信息不完整 | zip 默认不保存 Linux 的符号链接与特殊权限，跨系统还原会丢失 |
| 中文名编码 | Windows 打包的中文名多为 GBK，Linux 解压可能乱码，可试 unzip -O CP936 |

### 3.3 gzip 与 gunzip

gzip 只能压缩单个文件，默认压缩后删除原文件、生成 xxx.gz。

~~~bash
gzip app.log                 # 压缩为 app.log.gz，原文件被替换
gzip -k app.log              # -k 保留原文件
gzip -d app.log.gz           # 解压，等价于 gunzip app.log.gz
gunzip app.log.gz            # 与 gzip -d 等价的独立命令
gzip -l app.log.gz           # 查看压缩率，不解压
zcat app.log.gz              # 不解压直接查看内容
zcat app.log.gz | grep ERROR # 直接在压缩日志里检索
~~~

gzip 压缩目录会提示 is a directory，目录必须先用 tar 打包再压缩。

### 3.4 bzip2 与 xz

| 工具 | 压缩 | 解压 | 查看 | 选用建议 |
|---|---|---|---|---|
| bzip2 | bzip2 -k file | bunzip2 file.bz2 | bzcat file.bz2 | 压缩率高于 gzip、速度中等，后缀 .bz2 |
| xz | xz -k file | unxz file.xz | xzcat file.xz | 压缩率最高但最慢，适合归档分发，后缀 .xz |

配合 tar 时换成对应选项：-j 走 bzip2，-J 走 xz。

~~~bash
bzip2 -k app.log            # 生成 app.log.bz2 并保留原文件
tar -jcvf app.tar.bz2 app   # tar 加 bzip2
xz -k app.log               # 生成 app.log.xz 并保留原文件
tar -Jcvf app.tar.xz app    # tar 加 xz
tar -jtvf app.tar.bz2       # 查看 bzip2 压缩包内容
tar -Jtvf app.tar.xz        # 查看 xz 压缩包内容
~~~

选择顺序可以简单记为：只图快用 gzip，要兼顾体积用 bzip2，体积优先且不在乎耗时用 xz。

## 4. 解压安全与常见坑

### 4.1 路径穿越

压缩包里的条目路径不受限制，可能包含 ../ 或者以 / 开头的绝对路径。直接解压到 / 或 /usr 这类目录时，条目会被写到目标目录之外，覆盖系统文件，这类被刻意构造的包常被用来提权。所以来路不明的压缩包，解压前先看内容清单。

~~~bash
tar -tf unknown.tar.gz   # 列出 tar 包里的条目路径
unzip -l unknown.zip     # 列出 zip 包里的条目路径
~~~

清单里出现 ../、绝对路径，或者指向目录外的软链接时，不要直接解压到业务目录。稳妥做法是先解压到临时目录，确认无误再移动。

~~~bash
mkdir -p /tmp/inspect                    # 建临时目录
tar -xzf unknown.tar.gz -C /tmp/inspect  # 先解压到临时目录
find /tmp/inspect -type l                # 检查有没有可疑软链接
ls -R /tmp/inspect | head -30            # 大致浏览目录结构
mv /tmp/inspect/app /opt/app             # 确认无误后再移动到位
~~~

### 4.2 常见坑

| 坑 | 现象 | 规避方法 |
|---|---|---|
| 在 / 或 /usr 下随意解压 | 覆盖系统文件，严重时系统无法启动 | 一律解压到业务目录或临时目录 |
| 解压前不确认是否覆盖 | 同名文件被静默替换，旧版本丢失 | 先用 tar -tf 或 unzip -l 看清单，必要时先备份 |
| tar 的 -f 位置写错 | 归档名变成意外字符，命令卡住等输入 | f 放选项簇最后，后面紧跟归档文件名，用 ls 复核 |
| Windows 打包的中文名 | Linux 解压后文件名乱码 | 试 unzip -O CP936 指定编码，或改用 tar 包传输 |
| 解压后权限不对 | 脚本没有执行权限，服务启动失败 | tar 加 -p 保留权限，必要时手动 chmod +x |
| 用相对路径解压又忘了当前目录 | 文件散落在错误的目录 | 解压前 pwd 确认位置，或显式写 -C 指定目录 |

## 5. 本篇知识点总结

1. .tar 主要归档，.tar.gz 还使用 gzip 压缩。
2. tar 压缩常用 -cvf 或 -zcvf，解压常用 -xvf 或 -zxvf。
3. -f 指定归档文件，-C 指定解压目录。
4. zip 处理目录加 -r，unzip 用 -d 指定目标目录。
5. 打包与压缩是两件事，.tar.gz 才是先打包再压缩的组合。
6. 记忆法：打包 czvf、解压 xzvf、查看 tzvf，f 永远放最后并紧跟文件名。
7. -t 只查看内容清单，解压前先看清单是排查路径问题的第一步。
8. 跳过压缩：gzip -k 保留原文件，zcat 不解压直接查看内容。
9. bzip2 用 -j、xz 用 -J 配合 tar，压缩率越高通常速度越慢。
10. 压缩包可能含 ../ 路径，来路不明的包要先解压到临时目录再移动。
11. zip 默认不保留符号链接与特殊权限，跨平台传输权限信息会丢失。
12. 不要在 / 或 /usr 下随意解压，也不要覆盖同名文件前不做确认。
