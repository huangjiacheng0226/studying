# 03 Web 后端基础（Maven 基础）

Maven 是 Java 项目的事实标准构建工具，它负责下载依赖、统一目录结构、执行测试和打包。本篇从坐标、仓库和 `settings.xml` 讲起，覆盖依赖传递与冲突、生命周期命令，以及 JUnit 5 单元测试，最后补充多模块项目组织与本地仓库常见故障。

## 1. Maven 概述

### 1.1 Maven 解决的问题

Maven 用 POM 文件管理依赖、统一目录、执行测试、编译和打包。

POM 是 Maven 的项目对象模型；依赖通常先从本地仓库读取，缺少时再从远程仓库下载，插件负责编译、测试和打包等目标。默认构建输出目录是 target。

~~~text
src/main/java       主代码
src/main/resources  配置和资源
src/test/java       测试代码
pom.xml             项目描述文件
~~~

### 1.2 Maven 坐标

一个构件通常由 groupId、artifactId、version 唯一确定。

~~~xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
~~~

三段坐标的含义与命名规范如下：

| 坐标 | 含义 | 命名规范 | 示例 |
|---|---|---|---|
| `groupId` | 组织或公司标识，通常使用反向域名 | 全部小写，用点号分隔，一般不超过三段 | `com.example`、`org.springframework.boot` |
| `artifactId` | 项目或模块标识 | 全部小写，可用数字和连字符，见名知意 | `spring-boot-starter-web`、`user-service` |
| `version` | 版本号 | `主版本.次版本.修订号`；开发中的版本以 `-SNAPSHOT` 结尾 | `1.0.0`、`3.3.0`、`2.0.0-SNAPSHOT` |

坐标里不要出现中文和空格，也不要大小写混排。部分仓库和操作系统对文件名大小写敏感，命名不统一容易产生难以排查的路径问题。

坐标还会决定构件在仓库中的存放路径：`groupId` 的点号变成目录分隔符，然后是 `artifactId` 和 `version`，文件名由 `artifactId-version.扩展名` 组成。

~~~text
com/example/demo/1.0.0/demo-1.0.0.jar
~~~

`packaging` 决定构件打成什么类型，不写时默认 `jar`：

| packaging | 含义 | 常见场景 |
|---|---|---|
| `jar` | 普通 Java 归档，默认值 | 工具库、普通项目和 Spring Boot 内嵌服务器项目 |
| `war` | Web 归档，可以交给外部 Tomcat 部署 | 需要部署到外部 Tomcat 的传统 Web 项目 |
| `pom` | 不产出构件，只表达父子或聚合关系 | 父 POM、聚合工程 |

项目直接使用的依赖应直接声明，不要依赖偶然的传递依赖。

### 1.3 Maven 模型和仓库

Maven 项目由 `pom.xml` 描述，POM 中常见元素包括项目坐标、父工程、依赖、插件、属性和构建配置。仓库分为：

| 仓库 | 位置 | 作用 |
|---|---|---|
| 本地仓库 | 开发者电脑 | 缓存已下载的依赖和插件 |
| 中央仓库 | Maven 官方公共服务 | 提供常用开源构件 |
| 私服 | 公司内部服务器 | 缓存、审核和发布公司构件 |

依赖查找通常先检查本地仓库，本地没有时再从远程仓库下载。网络异常时可以检查仓库地址、代理和本地缓存。

### 1.4 安装和 IDEA 集成

安装 Maven 后配置 `MAVEN_HOME` 和 `PATH`，使用以下命令验证：

~~~text
mvn -v
~~~

IDEA 中可以在 Settings 的 Build Tools/Maven 中配置 Maven 主目录、用户配置文件和本地仓库。新手建议先使用项目统一的 Maven 配置，避免不同机器版本差异造成构建失败。

### 1.5 POM 常见配置

~~~xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>
  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
  </properties>
</project>
~~~

`packaging` 默认是 `jar`，Web 项目也可以使用 `war`。`properties` 用于统一 Java 版本、依赖版本和插件参数。父 POM 可以继承公共配置，子模块只保留自身依赖。

POM 还可以配置 `<build>`、插件和资源目录。插件决定“如何编译、测试和打包”，依赖决定“代码运行时需要哪些库”，二者不要混淆。

### 1.6 settings.xml 的三个关键配置段

`settings.xml` 是 Maven 的设置文件，它不描述某一个项目，而是描述“这台机器上的 Maven 怎么工作”。最常改动的三段是本地仓库、镜像和 JDK 版本。

第一段 `localRepository` 指定本地仓库目录，默认在用户目录下的 `.m2/repository`：

~~~xml
<settings>
  <localRepository>D:/maven/repository</localRepository>
</settings>
~~~

第二段 `mirrors` 指定镜像仓库，用来替换访问较慢的中央仓库。阿里云镜像的常见写法：

~~~xml
<mirrors>
  <mirror>
    <id>aliyunmaven</id>
    <name>阿里云公共仓库</name>
    <mirrorOf>central</mirrorOf>
    <url>https://maven.aliyun.com/repository/public</url>
  </mirror>
</mirrors>
~~~

`mirrorOf` 写 `central` 表示只在访问中央仓库时使用该镜像；如果写成 `*` 会镜像所有远程仓库，公司私服也会被顶替，通常不推荐。

第三段 `profiles` 用来约定 JDK 版本，可以同时设置编译版本属性和激活方式：

~~~xml
<profiles>
  <profile>
    <id>jdk-17</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <maven.compiler.source>17</maven.compiler.source>
      <maven.compiler.target>17</maven.compiler.target>
    </properties>
  </profile>
</profiles>
~~~

`maven.compiler.source` 和 `maven.compiler.target` 是 Maven 编译插件识别的属性，分别表示源码语言级别和目标字节码版本，两者一般保持一致。

`settings.xml` 可以放在两个位置，含义不同：

| 位置 | 作用范围 | 说明 |
|---|---|---|
| 用户目录下 `.m2/settings.xml` | 当前用户的全部项目 | 用户级配置，推荐修改这里，一人一份，互不影响 |
| Maven 安装目录下 `conf/settings.xml` | 使用该 Maven 安装的所有用户 | 全局级配置，适合统一内网机器 |

用户级与全局级同时存在时，用户级配置优先。安装目录下的 `conf/settings.xml` 是模板文件，升级 Maven 时可能被覆盖，不要随手修改。

### 1.7 IDEA 集成 Maven

IDEA 默认可能使用自己内置的 Maven，与命令行使用的版本不一致。在 `Settings → Build, Execution, Deployment → Build Tools → Maven` 中需要确认三处设置：

| 设置项 | 含义 | 常见取值 |
|---|---|---|
| Maven home path | 使用哪个 Maven 安装 | 本地安装目录，例如 `D:/maven/apache-maven-3.9.9` |
| User settings file | 读取哪个 `settings.xml` | 用户目录下 `.m2/settings.xml` |
| Local repository | 本地仓库目录 | 由 `settings.xml` 决定，一般不需要单独改 |

修改 `settings.xml` 后要回到这里确认 Local repository 是否跟着变化。如果依赖没有生效，点击 Maven 面板工具栏的 `Reload All Maven Projects` 重新导入；只改 POM 或配置文件不会自动刷新。

Runner 中的 VM Options 用于给 Maven 进程本身传递 JVM 参数，例如依赖下载或运行测试时内存不足：

~~~text
-Xmx2048m
~~~

这里填的是 Maven 运行参数，不是项目启动参数；Spring Boot 项目的启动参数应配置在对应的运行配置里。

## 2. 依赖管理

### 2.1 依赖范围

| scope | 编译 | 测试 | 运行 | 打包进构件 | 常见用途 |
|---|---|---|---|---|---|
| compile | 是 | 是 | 是 | 是 | 默认范围，绝大多数依赖 |
| provided | 是 | 是 | 否 | 否 | 容器或服务器已经提供，例如 Servlet API |
| runtime | 否 | 是 | 是 | 是 | 只在运行时需要，例如 JDBC 驱动 |
| test | 否 | 是 | 否 | 否 | JUnit、Mockito 等测试库 |
| system | 是 | 是 | 是 | 否 | 使用本机文件系统上的 jar，需要显式写 `systemPath`，不推荐 |
| import | 不参与 | 不参与 | 不参与 | 不参与 | 只能在 `dependencyManagement` 中导入 BOM |

列的含义：“编译”指主代码编译时是否进入类路径，“测试”指测试代码编译和运行测试时是否可用，“运行”指主程序运行时是否可用，“打包进构件”指是否被打进最终 jar 或 war。`provided` 的典型例子是 Servlet API：写代码时编译需要它，但运行时由 Tomcat 提供，打进 war 反而可能和容器自带的版本冲突。

`system` 需要配合 `systemPath` 指向本机某个 jar，换一台机器就可能构建失败，只应作为临时手段；`import` 不是普通依赖的范围，写错位置会直接报错。

传递依赖会自动引入间接库。版本冲突可用显式版本、dependencyManagement 或 exclusions 解决。

建议使用父 POM 或 dependencyManagement 统一版本，不要把 system scope 当作常规依赖方案。升级依赖后执行测试并用 dependency:tree 检查实际版本。

依赖冲突时 Maven 会根据依赖路径选择实际版本。可以在依赖树中定位冲突，再使用显式版本或 `<exclusions>` 排除不需要的传递依赖：

~~~xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>legacy-client</artifactId>
  <version>1.0.0</version>
  <exclusions>
    <exclusion>
      <groupId>org.example</groupId>
      <artifactId>old-lib</artifactId>
    </exclusion>
  </exclusions>
</dependency>
~~~

### 2.2 常用命令

~~~text
mvn clean
mvn test
mvn package
mvn install
mvn dependency:tree
~~~

`clean` 删除 target，`compile` 编译主代码，`test` 执行测试，`package` 生成 jar 或 war，`install` 将构件安装到本地仓库，`dependency:tree` 查看依赖树。

### 2.3 POM 依赖写法

~~~xml
<properties>
  <java.version>17</java.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
~~~

依赖的 `groupId`、`artifactId`、`version` 组成坐标。使用 Spring Boot 父 POM 或 BOM 时，很多版本可以由父工程统一管理。

`dependencyManagement` 只负责统一版本，不会自动把依赖加入项目；仍需在 `<dependencies>` 中声明实际使用的依赖。`import` 范围通常用于导入 BOM。

依赖排除只影响当前依赖路径，不会删除其他路径带来的同一构件。遇到 `ClassNotFoundException`、方法版本不匹配等问题，应先查看依赖树，再确认 scope 和实际打包内容。

### 2.4 依赖传递与冲突解决

项目直接写在 `<dependencies>` 中的依赖叫直接依赖；直接依赖自己又依赖的库叫间接依赖，也叫传递依赖。Maven 会把间接依赖一起下载并加入类路径，所以引入一个 starter 往往等于引入几十个 jar。

~~~text
项目
├── A 1.0（直接依赖，深度 1）
│   └── common 1.0（间接依赖，深度 2）
└── B 1.0（直接依赖，深度 1）
    └── common 2.0（间接依赖，深度 2）
~~~

上面这个例子里，`common` 有两个版本，Maven 只会选一个加入类路径。选择规则有两条：

1. 最短路径优先：依赖树的深度越小，优先级越高。如果 A 的路径是深度 2，另一个版本是深度 3，就用深度 2 的。
2. 路径长度相同时先声明优先：谁在 `pom.xml` 中写在前面，就用谁的版本。

本例中 `common` 的两条路径深度相同，此时取决于 A 和 B 谁先声明，A 先声明就用 `common 1.0`。

判断项目实际用了哪个版本，用依赖树查看：

~~~text
mvn dependency:tree
~~~

依赖冲突的常见解决方式：

| 方式 | 做法 | 适用场景 |
|---|---|---|
| 显式声明 | 在 `<dependencies>` 中直接写需要的版本，深度变成 1 | 只影响当前项目 |
| `dependencyManagement` | 在父工程统一锁定版本 | 多模块项目统一版本 |
| `<exclusions>` | 排除某个依赖带进来的间接依赖 | 明确不需要该库时 |

排除写法是在依赖内部加 `<exclusions>`（示例见 2.1），只需要写 `groupId` 和 `artifactId`，不需要写版本：

~~~xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>legacy-client</artifactId>
  <version>1.0.0</version>
  <exclusions>
    <exclusion>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
    </exclusion>
  </exclusions>
</dependency>
~~~

排除只对当前这一条依赖路径生效，其他路径仍可能把同一个构件带进来。排除项过多会让依赖树难以理解，优先考虑用统一版本管理代替到处排除。

## 3. 生命周期

Maven 有 clean、default、site 三条内置生命周期。执行后面的阶段，会先执行该生命周期的前置阶段。

常用 default 生命周期顺序是 `validate → compile → test → package → verify → install → deploy`。例如执行 `mvn package` 会先编译和测试，再生成可发布构件。

插件目标可以直接执行，例如 `mvn compiler:compile`、`mvn surefire:test`。实际项目通常通过生命周期绑定插件，不建议依赖某个开发者机器上的手动命令。

常用命令组合：`mvn clean package` 先清理旧产物再打包；`mvn test -DskipTests` 只编译测试但跳过执行（仅用于临时排错，不应作为质量验证）；`mvn package -DskipTests` 生成构件但跳过测试，发布前不要使用。

### 3.1 常用命令做什么、产物在哪里

前面提到的命令分属两条生命周期，执行后面的命令会自动执行同一生命周期的前置命令：

| 命令（阶段） | 做什么 | 产物位置 |
|---|---|---|
| `mvn clean` | 删除上一次构建的输出目录，属于 clean 生命周期 | 删除 `target/` |
| `mvn compile` | 编译 `src/main/java` 下的主代码 | `target/classes` |
| `mvn test` | 先编译主代码和测试代码，再执行测试 | `target/test-classes`、`target/surefire-reports` |
| `mvn package` | 先执行到 test，再把编译结果打成构件 | `target/*.jar` 或 `target/*.war` |
| `mvn install` | 先执行到 package，再把构件复制到本地仓库 | 本地仓库的 `groupId/artifactId/version/` 目录 |
| `mvn deploy` | 先执行到 install，再上传到远程私服或中央仓库 | 远程仓库 |
| `mvn dependency:tree` | 打印当前项目实际解析出的依赖树 | 控制台输出 |

所以 `mvn install` 已经包含了编译和测试，不需要先手动执行 `compile` 和 `test`。安装到本地仓库的路径就是把坐标展开成目录，例如 `com.example:demo:1.0.0` 会安装到：

~~~text
~/.m2/repository/com/example/demo/1.0.0/demo-1.0.0.jar
~~~

另外三点容易混淆：

- `mvn clean package` 是两条生命周期的命令连写，先清理再打包，不是因为 `package` 自带清理。
- `mvn deploy` 需要在 POM 中配置 `<distributionManagement>` 并有仓库权限，日常练习执行到 `install` 即可。
- `mvn -U clean package` 中的 `-U` 表示强制检查远程仓库的更新，依赖下载异常时常用。

## 4. JUnit 单元测试

~~~java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

class CalculatorTest {
    @Test
    void addsTwoNumbers() {
        assertEquals(5, 2 + 3);
    }
}
~~~

测试应独立、可重复，并使用断言验证结果。

常见 JUnit 5 注解：

| 注解 | 作用 | 说明 |
|---|---|---|
| `@Test` | 标记一个测试方法 | 方法不能有参数和返回值 |
| `@BeforeEach` | 每个测试方法执行前执行 | 适合准备测试数据 |
| `@AfterEach` | 每个测试方法执行后执行 | 适合释放资源 |
| `@BeforeAll` | 所有测试方法执行前执行一次 | 默认必须是 `static` 方法 |
| `@AfterAll` | 所有测试方法执行后执行一次 | 默认必须是 `static` 方法 |
| `@DisplayName` | 设置测试类或测试方法的可读名称 | 测试报告中显示中文名称 |
| `@Disabled` | 暂时跳过该测试 | 跳过的测试不计入失败 |
| `@ParameterizedTest` | 参数化测试，一个方法跑多组数据 | 需要数据来源注解 |
| `@ValueSource` | 提供一组简单类型的数据 | 配合 `@ParameterizedTest` 使用 |

为什么 `@BeforeAll` 和 `@AfterAll` 必须是 `static` 方法？因为它们要在任何测试实例创建之前或全部实例销毁之后执行，此时还没有可用的测试对象。

常见断言包括 `assertEquals`、`assertNotNull`、`assertTrue`、`assertThrows`。测试应只验证一个清晰行为，失败时能快速定位原因。

测试类通常放在 `src/test/java`，命名为 `XxxTest`。测试资源放在 `src/test/resources`。执行 `mvn test` 会由 Surefire 插件发现并运行测试；执行 `mvn verify` 可以在测试后执行更多质量检查。

单元测试应遵循 Arrange（准备）、Act（执行）、Assert（断言）结构。不要依赖测试执行顺序，也不要连接真实生产数据库；需要外部依赖时使用测试数据库、Mock 或独立测试容器。

### 4.1 为什么要有单元测试

单元测试针对最小的可测单元（通常是一个类或一个方法），不启动 Spring 容器、不连接数据库。它的价值有三个：改动代码后能立刻知道有没有把原有功能改坏；把需求中的边界情况固化成可重复执行的用例；发现缺陷时能定位到具体方法，而不是等到联调时才发现。

JUnit 5 的核心依赖坐标是：

~~~xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.9.1</version>
  <scope>test</scope>
</dependency>
~~~

`scope` 写 `test` 表示它只在编译测试代码和执行测试时可用，不会被打进最终构件。使用 `spring-boot-starter-test` 时其中已经包含 JUnit 5，只需要 JUnit 时用上面的坐标即可。

### 4.2 常用断言与参数化测试

| 断言 | 作用 |
|---|---|
| `Assertions.assertEquals(expected, actual)` | 判断两个值相等 |
| `Assertions.assertNotEquals` | 判断两个值不相等 |
| `Assertions.assertTrue` / `assertFalse` | 判断条件为真或为假 |
| `Assertions.assertNull` / `assertNotNull` | 判断对象是否为 null |
| `Assertions.assertThrows` | 判断是否抛出指定异常 |
| `Assertions.assertArrayEquals` | 判断两个数组内容是否相同 |

断言建议把期望值写在第一个参数、实际值写在第二个参数，失败信息才容易读。测试方法名用能说明行为的名字，例如 `addsTwoNumbers`、`throwsWhenBalanceIsNotEnough`。

参数化测试让同一个测试方法跑多组数据：

~~~java
class NumberTest {
    @ParameterizedTest
    @ValueSource(ints = {1, 3, 5, 7})
    @DisplayName("奇数判断")
    void shouldBeOdd(int number) {
        assertTrue(number % 2 == 1);
    }
}
~~~

一个测试方法只验证一个清晰的行为，断言过多会导致失败时定位变慢。

## 5. 前面章节小结

- Maven 管理依赖、构建和项目结构。
- 坐标、依赖范围、传递依赖和生命周期是重点。
- 遇到依赖问题先执行 dependency:tree。

## 6. Maven 高级项目组织

### 6.1 分模块设计

大型项目可以拆成 `pojo`、`mapper`、`service`、`web` 等模块。模块之间通过 Maven 依赖连接，职责清晰，修改和复用更方便；依赖方向应从上层业务模块指向下层公共模块，避免循环依赖。

### 6.2 继承与版本锁定

父 POM 用 `<parent>` 被子模块继承，适合统一 Java 版本、插件和依赖版本。`dependencyManagement` 只锁定版本，子模块仍要在 `<dependencies>` 中声明实际使用的依赖。

~~~xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.3.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
~~~

### 6.3 聚合工程

聚合工程通常只有一个父 POM，通过 `<modules>` 列出子模块。执行 `mvn clean install` 时，Maven 会按依赖顺序构建全部模块，不必手工逐个进入目录执行。

~~~xml
<packaging>pom</packaging>
<modules>
  <module>pojo</module>
  <module>service</module>
  <module>web</module>
</modules>
~~~

继承解决“配置复用”，聚合解决“统一构建”，一个工程可以同时使用二者。

### 6.4 私服和常见问题

私服用于缓存中央仓库依赖、发布公司内部构件和控制访问权限。上传/下载私服前要在 `settings.xml` 配置镜像、服务器账号和仓库地址；账号密码应使用环境变量或密码管理工具，不能提交到 Git。构建失败时优先查看完整错误堆栈、网络代理、JDK/Maven 版本和本地仓库缓存。

### 6.5 本地仓库与镜像的常见故障

依赖一直下载不下来，最常见的原因是本地仓库里残留了 `*.lastUpdated` 文件。

Maven 某次下载失败后，会在对应构件的目录下生成 `xxx.jar.lastUpdated` 之类的文件，记录这次失败的时间和原因。在 Maven 认定的重试时间窗内再次构建时会跳过该构件，于是表现为“怎么刷新都下不下来”，报错信息还停留在很久之前。

排查与清理步骤：

1. 在 IDEA 的 Maven 设置中确认本地仓库目录，打开该目录。
2. 停止正在进行的构建，删除出问题构件目录下的 `*.lastUpdated` 文件。
3. 检查 `settings.xml` 的镜像地址、网络和代理是否正常。
4. 在 IDEA 中执行 `Reload All Maven Projects`，或执行 `mvn -U clean package` 强制更新。

先查找本地仓库中的失败标记：

~~~bash
find . -name "*.lastUpdated"
~~~

确认范围后删除，例如只清理某个组织下的目录：

~~~bash
find ./com/example -name "*.lastUpdated" -delete
~~~

Windows 下也可以用 PowerShell 查找，把命令中的通配符交给 `-Filter`：

~~~text
Get-ChildItem -Recurse -Filter *.lastUpdated
~~~

镜像配置错误时有几种典型表现：

| 现象 | 可能原因 |
|---|---|
| 一直卡在 `Downloading` 或下载速度为 0 | 镜像地址不可达，或 `mirrorOf` 写法有误 |
| 报 `Could not transfer artifact ... Connection timed out` | 网络不通、代理未配置或被防火墙拦截 |
| 报 `Non-resolvable parent POM` | 父 POM 不在镜像覆盖范围内，或父工程坐标写错 |
| 配置了镜像却仍走中央仓库 | `mirrorOf` 没有匹配到目标仓库的 id |
| 依赖下载成功但版本不是预期的 | 本地仓库已有旧版本，或镜像内容滞后 |

改完 `settings.xml` 一定要重新加载 Maven 工程，配置文件不会自动生效。

## 7. 本章总结

- Maven 通过坐标、依赖和生命周期统一管理 Java 项目的构建过程。
- 依赖范围、传递依赖、冲突排除和 `dependencyManagement` 决定项目最终的类路径。
- 多模块项目用继承复用配置、用聚合统一构建；私服用于共享内部构件和缓存依赖。
- 日常排错从 `mvn dependency:tree`、完整错误堆栈和 JDK/Maven 版本开始。
- `groupId`、`artifactId`、`version` 唯一确定一个构件，`packaging` 决定打成 jar、war 还是只做父工程。
- `settings.xml` 中的本地仓库、镜像和 JDK profile 决定本机 Maven 的行为，用户级配置优先于安装目录下的全局配置。
- 依赖冲突遵循最短路径优先、路径相同时先声明优先，可用显式版本、`dependencyManagement` 或 `<exclusions>` 处理。
- 单元测试只需要 `org.junit.jupiter:junit-jupiter` 且 scope 为 test，测试类放在 `src/test/java`。
- 本地仓库里的 `*.lastUpdated` 是下载失败标记，删除后重新刷新即可恢复下载。
