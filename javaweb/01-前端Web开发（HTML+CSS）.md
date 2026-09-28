# 01 前端 Web 开发（HTML + CSS）

## 1. Web 前端与 Web 标准

### 1.1 前端职责

前端负责把文字、图片、表格、表单等数据组织成用户可见的页面。浏览器会解析 HTML、CSS 和 JavaScript，再通过渲染引擎显示结果。

### 1.2 技术分工

| 技术 | 主要职责 | 示例 |
|---|---|---|
| HTML | 页面结构和内容 | 标题、表格、表单 |
| CSS | 页面外观和布局 | 颜色、间距、Flex |
| JavaScript | 页面行为和交互 | 点击、校验、异步请求 |

### 1.3 Web 标准和浏览器工作方式

Web 标准主要由结构、表现和行为三部分组成：HTML 描述内容结构，CSS 描述外观，JavaScript 负责交互行为。分离三者可以减少重复代码，便于维护和无障碍访问。

浏览器访问页面时会先解析 HTML 生成 DOM 树，再解析 CSS 生成 CSSOM，合并后计算布局并绘制页面；脚本可能修改 DOM，从而触发重新布局或重绘。因此脚本应放在合适位置，避免频繁读写布局属性。

### 1.4 前后端分离与前后端混合开发

前后端混合开发指后端使用模板引擎（如 JSP、Thymeleaf）生成 HTML，页面结构和数据在同一份代码里；前后端分离指前端是独立工程，通过 Ajax 调用后端提供的 JSON 接口拿数据，自己负责渲染。

| 对比项 | 前后端混合开发 | 前后端分离 |
|---|---|---|
| 开发方式 | 后端写模板，页面由服务端渲染 | 前端独立开发页面，后端只提供接口 |
| 代码组织 | 页面与后端代码在同一工程，耦合较高 | 前后端各自独立工程，按接口契约协作 |
| 数据传递 | 后端把数据放进模型，模板中直接使用 | 前端发 Ajax 请求，后端返回 JSON，前端渲染 |
| 部署 | 打包成一个应用一起部署 | 前端静态资源由 Nginx/CDN 托管，后端单独部署 API |
| 优点 | 首屏渲染快、对搜索引擎友好、接口暴露少 | 职责清晰、可并行开发、前端可复用、便于多端 |
| 缺点 | 前后端耦合，前端调试依赖后端环境 | 需要处理跨域、接口约定与鉴权，首屏依赖请求 |

两种方式的请求过程不同：

~~~mermaid
flowchart LR
    subgraph 混合开发
        A1[浏览器请求页面] --> A2[后端渲染模板]
        A2 --> A3[返回完整 HTML]
    end
    subgraph 前后端分离
        B1[浏览器请求页面] --> B2[返回静态 HTML 与 JS]
        B2 --> B3[前端发 Ajax 请求接口]
        B3 --> B4[后端返回 JSON]
        B4 --> B5[前端渲染数据]
    end
~~~

Tlias 项目采用前后端分离方案：前端页面通过 Axios 调用后端接口，后端只返回 JSON，页面由前端渲染。

## 2. HTML 基础

### 2.1 页面骨架

~~~html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>部门管理</title>
</head>
<body>
  <h1>部门列表</h1>
</body>
</html>
~~~

head 保存标题、元数据和样式，body 保存用户可见内容。标签建议小写，属性值使用双引号。

### 2.2 常用标签

| 场景 | 标签 | 说明 |
|---|---|---|
| 标题 | h1 到 h6 | 数字越小级别越高 |
| 段落 | p | 表示一段文字 |
| 换行 | br | 插入换行 |
| 链接 | a | href 指定地址 |
| 图片 | img | src 指定路径，alt 提供替代文本 |
| 视频/音频 | video、audio | controls 显示控件 |
| 布局 | div、span | 块级和行内容器 |
| 表格 | table、tr、th、td | 表头使用 th |
| 表单 | form、input、select、button | name 影响提交参数 |

页面主内容应优先使用 header、nav、main、section、footer 等语义标签；表单控件配合 label，图片填写有意义的 alt，并保证交互控件可以通过键盘 Tab 访问。

表格应使用 caption 说明用途，表头使用 th 并通过 scope 标明行列关系：

~~~html
<table>
  <caption>部门列表</caption>
  <thead><tr><th scope="col">编号</th><th scope="col">名称</th></tr></thead>
  <tbody><tr><td>1</td><td>教研部</td></tr></tbody>
</table>
~~~

### 2.3 表单示例

~~~html
<form id="deptForm">
  <label for="name">部门名称</label>
  <input id="name" name="name" maxlength="10" required>
  <select name="status">
    <option value="1">启用</option>
    <option value="0">禁用</option>
  </select>
  <button type="submit">保存</button>
</form>
~~~

### 2.4 路径和常用属性

路径分三类，容易混淆的是以 `/` 开头的那种：相对路径以当前 HTML 文件所在目录为基准；以 `/` 开头的是相对服务器根目录的路径（根路径），它不是本地文件系统的绝对路径，也不包含应用的上下文路径；完整 URL（如 `https://example.com/img/a.png`）才是通常所说的绝对地址。

以 `/` 开头最常见的坑是忽略上下文路径。假设应用部署后访问地址是 `http://host/tlias/index.html`，页面里写 `/css/style.css`，浏览器会去请求 `http://host/css/style.css`，绕过了 `/tlias`，静态资源往往直接变成 404。

~~~html
<!-- 理解错误：把 / 当成磁盘根目录，或忽略应用上下文路径 -->
<link rel="stylesheet" href="/css/style.css">

<!-- 正确：相对当前页面，部署到哪个目录都不会错 -->
<link rel="stylesheet" href="css/style.css">

<!-- 正确：确实需要根路径时，显式带上上下文路径 -->
<link rel="stylesheet" href="/tlias/css/style.css">

<!-- 正确：引用外部资源时使用完整 URL -->
<script src="https://cdn.example.com/vue.global.js"></script>

<!-- 错误：不能把本机磁盘路径写进页面 -->
<img src="D:\project\css\logo.png" alt="logo">
~~~

项目中图片、CSS 和脚本通常使用相对路径，既避免把本机磁盘路径写入页面，也不会因为上下文路径变化而失效；接口地址则建议由前端统一配置，不要散落在每个页面里。

| 属性 | 适用标签 | 作用 |
|---|---|---|
| `id` | 任意元素 | 唯一标识元素 |
| `class` | 任意元素 | 关联一个或多个 CSS 类 |
| `href` | `a`、`link` | 指定链接或资源地址 |
| `src` | `img`、`script` | 指定外部资源地址 |
| `alt` | `img` | 图片无法显示时的替代文本 |
| `name` | 表单控件 | 作为提交参数名 |
| `value` | 表单控件 | 控件提交的值 |

### 2.5 表格和表单细节

表格通常由 `table`、`caption`、`thead`、`tbody`、`tr`、`th` 和 `td` 组成。`th` 表示表头，`scope="col"` 表示列标题，`scope="row"` 表示行标题。不要使用表格完成页面整体布局。

表单常见控件包括文本框、密码框、单选框、复选框、下拉框、文件框和按钮：

~~~html
<form action="/depts" method="post">
  <input name="name" type="text" required maxlength="10">
  <input name="status" type="radio" value="1" checked>启用
  <input name="status" type="radio" value="0">禁用
  <button type="submit">保存</button>
</form>
~~~

`label for` 应与控件的 `id` 对应；提交按钮使用 `type="submit"`，普通按钮使用 `type="button"`，避免误提交表单。

表单提交时只有具有 `name` 的控件会生成参数；单选框同名时只能选一个，复选框同名时可能提交多个值。`GET` 会把参数拼到 URL，`POST` 通常把参数放进请求体；前端校验不能代替后端校验。

### 2.6 表单控件：input 类型、文本域、下拉框与 label

`input` 是最常用的表单控件，它的外观和取值方式由 `type` 决定。表单提交时，只有带 `name` 的控件才会生成参数。

| type | 作用 | 是否参与提交 | 说明 |
|---|---|---|---|
| `text` | 单行文本框 | 有 `name` 才提交 | 默认类型，可用 `maxlength` 限制长度 |
| `password` | 密码框 | 有 `name` 才提交 | 输入内容显示为圆点，但传输时不加密 |
| `radio` | 单选框 | 有 `name` 才提交 | 同名一组只能选一个，必须写 `value` |
| `checkbox` | 复选框 | 有 `name` 才提交 | 同名可以提交多个值 |
| `file` | 文件选择框 | 有 `name` 才提交 | 表单需要 `enctype="multipart/form-data"` |
| `date` | 日期选择框 | 有 `name` 才提交 | 提交格式为 `YYYY-MM-DD` |
| `number` | 数字输入框 | 有 `name` 才提交 | 可用 `min`、`max`、`step` 限制范围和步长 |
| `hidden` | 隐藏域 | 有 `name` 才提交 | 用户看不到，适合传递 id 等隐藏数据 |
| `submit` | 提交按钮 | 不生成参数 | 点击后提交表单；带 `name` 时按钮自身的值也会提交 |
| `reset` | 重置按钮 | 不生成参数 | 把控件恢复到默认值，不提交表单 |
| `button` | 普通按钮 | 不生成参数 | 没有默认行为，通常配合 JavaScript 使用 |

~~~html
<form action="/depts" method="post" enctype="multipart/form-data">
  <input type="text" name="name" placeholder="请输入部门名称">
  <input type="password" name="password">
  <input type="radio" name="status" value="1" checked>启用
  <input type="radio" name="status" value="0">禁用
  <input type="checkbox" name="roles" value="admin">管理员
  <input type="file" name="avatar">
  <input type="date" name="hiredate">
  <input type="number" name="sort" min="1" max="10" step="1">
  <input type="hidden" name="id" value="1">
  <input type="submit" value="保存">
  <input type="reset" value="重置">
  <input type="button" value="取消" onclick="cancelEdit()">
</form>
~~~

多行文本使用 `textarea`，它的初始值写在标签之间，而不是 `value` 属性：

~~~html
<textarea name="remark" rows="3" cols="20">默认内容</textarea>
~~~

下拉框使用 `select` 包裹若干 `option`，每个可选项用 `value` 表示真正提交的值：

~~~html
<select name="deptId">
  <option value="">请选择</option>
  <option value="1" selected>教研部</option>
  <option value="2">学工部</option>
</select>
~~~

下拉框提交的是被选中 `option` 的 `value`；不写 `value` 时会退回提交 `option` 的文本内容。给 `select` 加上 `multiple` 属性可以多选，用 `size` 可以设置一次显示几行。

`label` 用来描述控件，把文字和控件关联后，点击文字也能聚焦或选中对应控件。两种写法：

~~~html
<label for="deptName">部门名称</label>
<input id="deptName" name="name">

<label>
  部门名称
  <input name="name">
</label>
~~~

`for` 的值必须等于控件的 `id`，这是推荐写法；`label` 还能让屏幕阅读器正确朗读控件含义，提升可访问性。

### 2.7 字符实体与文本语义标签

HTML 里的尖括号、与号等字符会被当作标签或语法的开始，想在页面上把这些符号本身显示出来，就要使用字符实体：以 `&` 开头、以 `;` 结尾的一段编码。

| 字符实体 | 显示结果 | 用途 |
|---|---|---|
| `&lt;` | `<` | 小于号，展示标签写法时必用 |
| `&gt;` | `>` | 大于号 |
| `&amp;` | `&` | 与号本身 |
| `&nbsp;` | 一个不换行空格 | HTML 会把连续空格合并成一个，需要多个空格时使用 |
| `&quot;` | `"` | 双引号，出现在属性值中时使用 |
| `&copy;` | 版权符号 | 页脚版权声明 |

~~~html
<p>&lt;div&gt; 是块级标签，&amp; 是字符实体的起始符</p>
<p>价格：&yen;99&nbsp;&nbsp;元，&copy; 2024 Tlias</p>
~~~

`&nbsp;` 是不换行空格，浏览器不会在它所在位置折行，所以不要用它来做段落缩进，缩进应该交给 CSS 的 `text-indent`。

HTML 中有些标签外观相似但语义不同，应该按含义选择标签，而不是按“看起来加粗/倾斜”选择：

| 标签 | 默认外观 | 语义 | 使用建议 |
|---|---|---|---|
| `b` | 加粗 | 无语义，只是视觉加粗 | 关键词、产品名等不需要强调语义的场景 |
| `strong` | 加粗 | 表示内容重要 | 语气上需要强调的重要内容，屏幕阅读器可能加重朗读 |
| `i` | 倾斜 | 无语义，只是视觉倾斜 | 图标字体、术语、外语单词 |
| `em` | 倾斜 | 表示强调 | 句子中需要强调的词 |
| `u` | 下划线 | 无语义 | 谨慎使用，容易和链接混淆，可能被当成拼写错误 |
| `s` | 删除线 | 无语义 | 表示内容不再准确，如旧价格 |
| `del` | 删除线 | 表示文档中被删除的内容 | 修订类文本，常与表示新增的 `ins` 搭配 |

~~~html
<p>旧价格：<del>&yen;199</del>，现价：<strong>&yen;99</strong></p>
<p>这是一段 <em>非常重要</em> 的提示</p>
~~~

实际项目中优先使用 `strong` 和 `em`，只有在确实不需要语义时才用 `b` 和 `i`。

## 3. CSS 样式

### 3.1 三种引入方式

| 方式 | 特点 | 建议 |
|---|---|---|
| 行内样式 | 写在 style 属性中 | 临时演示 |
| 内部样式 | 写在 style 标签中 | 单页面小项目 |
| 外部样式 | link 引入 CSS 文件 | 正式项目首选 |

### 3.2 选择器

~~~css
* { box-sizing: border-box; }

/* class 选择器可复用 */
.primary { color: #1677ff; }

/* id 选择器通常只使用一次 */
#app { max-width: 960px; margin: 0 auto; }
~~~

选择器优先级大致为：行内样式 > id > class/属性/伪类 > 元素。项目中优先使用 class。

### 3.3 常见选择器和伪类

~~~css
/* 元素、类、ID、后代、子元素选择器 */
p { line-height: 1.6; }
.card { padding: 16px; }
#app { min-height: 100vh; }
.card .title { font-weight: 600; }
.menu > li { list-style: none; }

/* 伪类 */
a:hover { color: #1677ff; }
input:focus { outline: 2px solid #1677ff; }
li:first-child { font-weight: bold; }
~~~

选择器越具体，优先级通常越高；项目中应避免层级过深和大量 `!important`，优先通过清晰的 class 设计样式。

### 3.4 CSS 文本样式

文本样式主要控制缩进、行高、对齐、装饰线和颜色，是还原设计稿时最先用到的一组属性。

| 属性 | 作用 | 常用值 |
|---|---|---|
| `text-indent` | 首行缩进 | `2em`，表示缩进两个字符 |
| `line-height` | 行高，即行与行之间的距离 | 不带单位的数字（如 `1.6`）或 `px` |
| `text-align` | 水平对齐 | `left`、`center`、`right`、`justify` |
| `text-decoration` | 文本装饰线 | `none`、`underline`、`line-through` |
| `color` | 文字颜色 | 颜色关键字、`#1677ff`、`rgb()`、`rgba()` |
| `font-size` | 字号 | `14px`、`16px` 等 |

~~~css
.article {
  text-indent: 2em;
  line-height: 1.6;
  text-align: justify;
  color: #333333;
  font-size: 16px;
}

.nav a {
  text-decoration: none;
}
~~~

`text-indent` 只影响第一行，单位常用 `em`，这样字号变化时缩进会跟着变。`line-height` 写成不带单位的数字时，浏览器用“数字 × 字号”计算行高，后期改字号也不会打乱间距，因此比固定 `px` 更常用。

单行文本垂直居中的常见技巧是让 `line-height` 等于盒子高度，再加 `text-align: center` 实现水平居中：

~~~css
.btn {
  height: 40px;
  line-height: 40px;
  text-align: center;
}
~~~

颜色有几种写法：颜色关键字（如 `red`、`black`）、十六进制 `#1677ff`、`rgb(22, 119, 255)`，以及带透明度的 `rgba(22, 119, 255, 0.5)`。十六进制是设计稿最常给出的形式，`rgba()` 适合做半透明遮罩。

`text-decoration: none` 最常用于去掉链接默认的下划线；`line-through` 表示删除线，常用于旧价格。正文行高一般取 `1.5` 到 `1.8`，行高过小会让相邻两行挤在一起。

### 3.5 盒子模型和 Flex

盒子由 content、padding、border、margin 组成。box-sizing: border-box 会把内边距和边框计入宽高。

~~~css
.layout {
  display: flex;
  gap: 16px;
  align-items: center;
  justify-content: space-between;
}
.panel {
  width: 320px;
  padding: 16px;
  border: 1px solid #ddd;
  margin: 8px;
}
~~~

`padding` 是内边距，`margin` 是外边距，两者都可以写 1 到 4 个值，浏览器按“上、右、下、左”的顺序补全：

| 写法 | 含义 |
|---|---|
| `padding: 10px` | 四边都是 10px |
| `padding: 10px 20px` | 上下 10px，左右 20px |
| `padding: 10px 20px 30px` | 上 10px，左右 20px，下 30px |
| `padding: 10px 20px 30px 40px` | 上、右、下、左（顺时针） |
| `margin: 10px 0` | 上下 10px，左右 0 |
| `margin: 0 auto` | 上下 0，左右自动平分 |

简写规则可以这样记：两个值表示上下和左右，三个值表示上、左右、下，四个值按顺时针依次是上、右、下、左。

`margin: 0 auto` 让块级元素水平居中的前提是元素本身有固定宽度（或 `max-width`）。如果元素宽度是默认的 `auto`，它会撑满父容器，居中就看不出效果：

~~~css
.container {
  width: 960px;
  margin: 0 auto;
}
~~~

`padding` 撑开盒子内部空间，不能为负值；`margin` 在盒子外部，可以为负值，会让元素相互靠拢。在未使用 `box-sizing: border-box` 时，`padding` 和 `border` 会额外加大元素占用的空间，容易造成“设置了 100% 宽度却溢出”的问题。

垂直方向相邻的外边距会发生合并，最终取两者中较大的值，而不是相加：

~~~css
.a { margin-bottom: 30px; }
.b { margin-top: 20px; }
/* a 与 b 相邻时，间距是 30px，不是 50px */
~~~

父子元素之间也会合并：父元素没有 `padding`、`border`、`overflow: hidden` 等隔离条件时，子元素的上外边距会“漏”到父元素外面，表现为父元素被顶下来。常见处理办法是给父元素加 `padding` 或 `border`，或加 `overflow: hidden` 触发 BFC。

Flex 适合一维布局，Grid 适合二维布局。移动端应使用相对单位和媒体查询。

### 3.6 常见布局和响应式

`display: block` 元素独占一行，`inline` 元素不能设置完整宽高，`inline-block` 兼具两者部分特征。Flex 常用属性如下：

| 属性 | 作用 |
|---|---|
| `display: flex` | 启用 Flex 布局 |
| `flex-direction` | 设置主轴方向 |
| `justify-content` | 设置主轴对齐 |
| `align-items` | 设置交叉轴对齐 |
| `flex-wrap` | 是否换行 |
| `gap` | 子元素间距 |

响应式页面可以使用媒体查询：

~~~css
.content { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }

@media (max-width: 768px) {
  .content { grid-template-columns: 1fr; }
}
~~~

### 3.7 CSS 单位和继承

`px` 是固定像素，`%` 相对父元素，`rem` 相对根元素字体大小，`vw` 和 `vh` 相对视口。字体、颜色等属性可以继承，盒模型尺寸通常不会自动继承。

### 3.8 定位、层叠和常见排错

`position: relative` 保留原位置并提供定位参照，`absolute` 脱离普通文档流，`fixed` 相对视口固定，`sticky` 在滚动到阈值后固定。多个元素重叠时可用 `z-index` 调整层叠顺序，但只有定位元素或 Flex/Grid 子项等场景才生效。

页面显示不符合预期时，优先在浏览器开发者工具中检查元素、Computed 样式、盒模型和 Network 请求；常见原因是选择器优先级、继承、路径错误、元素默认 margin 或宽度超出父容器。

## 4. Tlias 案例静态页面

Tlias 前端开发的第一步是把设计稿还原成不接数据的静态页面，这一节的目的是把前面学到的标签、路径、盒子模型和布局串起来。

### 4.1 页面由哪些区块组成

以部门管理和员工管理页面为例，页面可以拆成下面几个区块：

| 区块 | 组成 | 主要标签 | 布局要点 |
|---|---|---|---|
| 顶部导航 | logo、系统名称、当前用户、退出 | `header`、`nav`、`img`、`a` | 固定高度，横向排列 |
| 左侧菜单 | 部门管理、员工管理、班级管理 | `aside`、`ul`、`li`、`a` | 固定宽度，深色背景 |
| 主内容区 | 页面标题、操作按钮、表格、分页 | `main`、`section`、`table`、`button` | 宽度自适应，`flex: 1` |
| 搜索栏 | 输入框、日期选择、查询与重置按钮 | `form`、`label`、`input`、`select` | Flex 横向排列，`gap` 控制间距 |
| 新增/编辑弹窗 | 表单、确定与取消按钮 | `div` 加定位、`form`、`input`、`button` | `position: fixed` 加遮罩层 |

先用语义标签搭出结构，再用 CSS 控制外观：

~~~html
<body>
  <header class="top-bar">Tlias 智能学习辅助系统</header>
  <div class="layout">
    <aside class="menu">
      <ul>
        <li><a href="#dept">部门管理</a></li>
        <li><a href="#emp">员工管理</a></li>
      </ul>
    </aside>
    <main class="content">
      <h2>部门管理</h2>
      <form class="search-bar">
        <label for="deptName">部门名称</label>
        <input id="deptName" name="name" type="text" placeholder="请输入部门名称">
        <button type="submit">查询</button>
      </form>
      <table>
        <caption>部门列表</caption>
        <thead>
          <tr><th scope="col">编号</th><th scope="col">名称</th><th scope="col">操作</th></tr>
        </thead>
        <tbody>
          <tr><td>1</td><td>教研部</td><td><a href="#">编辑</a></td></tr>
        </tbody>
      </table>
    </main>
  </div>
</body>
~~~

### 4.2 用哪些 CSS 实现布局

顶部固定、左侧固定宽度、右侧自适应，是典型的两栏布局，用 Flex 实现最简单：

~~~css
.layout {
  display: flex;
  min-height: calc(100vh - 60px);
}

.menu {
  width: 200px;
  background: #001529;
}

.content {
  flex: 1;
  padding: 16px;
}

.search-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}
~~~

表格用 `border-collapse: collapse` 去掉单元格之间的缝隙，表头设置背景色，单元格用 `padding` 控制留白：

~~~css
table {
  width: 100%;
  border-collapse: collapse;
}

th,
td {
  padding: 8px 12px;
  border-bottom: 1px solid #eeeeee;
  text-align: left;
}
~~~

按钮、输入框、分页等重复出现的元素统一抽成 class，避免每个元素写一遍样式；弹窗用 `position: fixed` 加半透明遮罩层实现，不要用表格或大量 `br` 去排版。

### 4.3 用开发者工具调试

页面效果和设计稿不一致时，按固定顺序排查，比盲目改 CSS 效率高：

~~~mermaid
flowchart TD
    A[页面效果不符合预期] --> B[用 Elements 面板选中元素]
    B --> C[查看 Computed 与盒模型]
    C --> D{样式被覆盖?}
    D -- 是 --> E[看被划掉的规则来自哪个选择器, 调整优先级]
    D -- 否 --> F{尺寸或位置不对?}
    F -- 是 --> G[检查 width, padding, margin 与 box-sizing]
    F -- 否 --> H[用 Network 面板确认资源是否 404]
    H --> I[修正路径后重新验证]
~~~

常用面板的分工：

| 面板 | 用途 |
|---|---|
| Elements | 查看 DOM 结构、临时改样式、查看元素上生效和被覆盖的 CSS |
| Computed | 查看最终计算出的尺寸和样式，定位继承或被覆盖的属性 |
| Network | 查看静态资源是否 404、接口返回了什么数据 |
| Device Toolbar | 切换手机尺寸，验证响应式效果 |

在 Elements 面板里直接修改样式只对当前页面生效，刷新就会丢失；确认原因后仍然要回到 CSS 文件里修改，再刷新验证一次。

## 5. 小白易错点

- 把以 `/` 开头的路径当成“绝对路径”或磁盘根目录，忽略应用的上下文路径，静态资源因此 404；相对路径 `css/style.css` 才是最稳的写法。
- 把本机路径（如 `D:\project\css\logo.png`）写进 `src`、`href`，本机预览正常，部署后一定失效。
- 表单控件忘了写 `name`，提交时没有任何参数，后端收到空值。
- 单选框同名一组却写成了不同的 `name`，结果能同时选中多个。
- `label` 的 `for` 和控件 `id` 不一致，点击文字无法聚焦控件。
- 想用连续空格排版，写了多个空格却被 HTML 合并成一个，正确做法是用 CSS 的 `text-indent` 或 `padding`。
- 用 `b`、`i` 代替 `strong`、`em`，丢掉了语义；反向用 `strong` 包一整套排版标题也不合适。
- `margin: 0 auto` 不生效，通常是因为元素没有固定宽度，或元素不是块级元素。
- 两个相邻元素设置上下 `margin` 后间距比预期小，这是外边距合并取较大值，不是计算错误。
- 用 `text-align: center` 给块级元素本身居中，实际只居中了元素内部的文本，元素居中要用 `margin: 0 auto` 或 Flex。

## 6. 练习清单

1. 为一个部门管理页面写出顶部导航、左侧菜单、主内容区的 HTML 结构，使用语义标签。
2. 用 `text-indent`、`line-height`、`text-align`、`text-decoration` 还原一段新闻正文样式。
3. 做出一个包含文本框、单选框、复选框、下拉框、日期框和文件框的表单，并用 `label` 绑定每个控件。
4. 用两种颜色值写法（十六进制和 `rgba`）分别设置文字颜色和半透明遮罩。
5. 用 `width` 加 `margin: 0 auto` 实现一个 960px 的居中容器，再用 Flex 实现顶部导航与内容区的两栏布局。
6. 故意写出一个使用 `/css/style.css` 导致 404 的页面，用开发者工具的 Network 面板定位问题并修正。
7. 在页面上输出 `&lt;div&gt;` 和 `&amp;`，确认浏览器把它们显示成标签写法和与号本身。

## 7. 资料对应关系

- 《JavaWeb笔记》：第一章“前端 Web 开发（HTML + CSS）”。
- MDN HTML 元素参考：[HTML 元素](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element)。
- MDN CSS 参考：[CSS 参考](https://developer.mozilla.org/zh-CN/docs/Web/CSS)。
- 下一章 [02-前端Web开发（JS+Vue+Ajax）.md](02-前端Web开发（JS+Vue+Ajax）.md) 讲解页面行为与异步请求。

## 8. 本章总结

- HTML 负责结构，CSS 负责表现。
- 重点掌握标签、路径、表单、选择器和盒子模型。
- 正式项目优先使用外部 CSS 和可复用 class。
