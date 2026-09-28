# 02 前端 Web 开发（JavaScript + Vue + Ajax）

## 1. JavaScript 核心语法

### 1.1 引入方式

~~~html
<script src="app.js" defer></script>
~~~

外部 JS 文件只写 JavaScript，不要再写 script 标签。defer 让脚本在 HTML 解析完成后执行。

### 1.2 变量、类型和函数

推荐用 let 声明变量，用 const 声明常量，避免使用 var。

~~~javascript
const apiBase = '/api';
let count = 0;
count += 1;

function sum(a, b) {
  return a + b;
}
~~~

常见类型：string、number、boolean、undefined、null；数组和对象属于引用类型。

#### 1.2.1 输出和运算

~~~javascript
console.log('控制台输出');
document.write('<p>页面输出</p>');
alert('弹窗提示');

const total = 10 + 2 * 3;
const ok = total >= 10 && total < 20;
~~~

JavaScript 的常见运算符包括算术运算符、比较运算符、逻辑运算符和赋值运算符。比较值和类型时优先使用严格相等 `===`，避免隐式类型转换。

#### 1.2.2 类型和类型转换

| 类型 | 示例 | 说明 |
|---|---|---|
| string | `'Tom'` | 字符串 |
| number | `18`、`3.14` | 整数和小数统一为 number |
| boolean | `true`、`false` | 布尔值 |
| undefined | `let value` | 已声明但没有值 |
| null | `null` | 明确表示没有对象 |
| object | `{ id: 1 }` | 对象、数组、日期等 |

表单输入值通常是字符串，需要时使用 `Number(value)`、`String(value)` 或 `Boolean(value)` 转换。

变量名使用字母、数字、`_` 或 `$`，不能以数字开头；常量通常使用全大写加下划线。`typeof null` 的结果是 `"object"`，判断数组应使用 `Array.isArray(value)`。`==` 会先进行隐式转换，初学阶段优先使用 `===` 和 `!==`。

### 1.3 流程控制和函数

~~~javascript
function findMax(a, b) {
  if (a > b) {
    return a;
  }
  return b;
}

for (let i = 0; i < 3; i += 1) {
  console.log(i);
}

const names = ['Tom', 'Jerry'];
names.forEach(name => console.log(name));
~~~

函数可以使用声明式、表达式或箭头函数定义。参数是调用者传入的数据，`return` 把结果交给调用者。数组常用 `push`、`pop`、`slice`、`map`、`filter` 和 `find` 方法。

### 1.4 对象、数组、字符串和 JSON

~~~javascript
const user = { id: 1, name: 'Tom' };
user.name = 'Jerry';

const jsonText = JSON.stringify(user);
const copy = JSON.parse(jsonText);
const message = `用户：${copy.name}`;
~~~

对象使用键值对保存数据，数组保存有序集合。JSON 只能表示数据，不能保存函数；前后端传输 JSON 时，发送端使用 `JSON.stringify`，接收端使用 `response.json()` 或 `JSON.parse`。

### 1.5 DOM 与事件

~~~javascript
const button = document.querySelector('#loadBtn');

button.addEventListener('click', () => {
  document.querySelector('#message').textContent = '加载中...';
});
~~~

推荐使用 addEventListener，普通文本使用 textContent 更新，避免不可信输入直接写入 innerHTML。

BOM 表示浏览器窗口、地址和历史记录等对象；DOM 把 HTML 文档表示为内存中的树。事件通常经历捕获、目标、冒泡三个阶段，需要时可使用 event.stopPropagation() 阻止继续传播。

常用 DOM API：

| API | 作用 |
|---|---|
| `querySelector` | 获取第一个匹配元素 |
| `querySelectorAll` | 获取全部匹配元素 |
| `createElement` | 创建元素 |
| `append` | 添加子节点 |
| `classList.add` | 添加 CSS 类 |
| `addEventListener` | 注册事件监听器 |

事件传播方向是捕获阶段、目标阶段、冒泡阶段。父元素监听点击时，子元素点击可能触发父元素处理；需要阻止时调用 `event.stopPropagation()`。

#### 1.5.1 addEventListener 与 onclick

以 `on` 开头的属性（`onclick`、`onchange` 等）是事件属性，同一个事件只能挂一个处理函数；`addEventListener` 可以挂多个处理函数，也能精确移除指定的那个。

| 对比项 | `onclick` 属性 | `addEventListener` |
|---|---|---|
| 绑定方式 | `element.onclick = fn`，或 HTML 里写 `onclick="save()"` | `element.addEventListener('click', fn)` |
| 能否绑定多个 | 不能，后写的会覆盖前面的 | 可以，按注册顺序依次执行 |
| 能否移除 | 只能整体清空：`element.onclick = null` | `removeEventListener('click', fn)` 移除指定函数 |
| 事件类型 | 每种事件只有一个固定的 `onxxx` 属性 | 任意事件名，可用变量传参 |
| 捕获阶段 | 不支持指定 | 第三个参数传 `true` 时在捕获阶段触发 |
| 可选参数 | 无 | 可传 `{ once: true }`、`{ passive: true }` 等 |

~~~javascript
const button = document.querySelector('#saveBtn');

// onclick 只会保留最后一个处理函数
button.onclick = () => console.log('第一次');
button.onclick = () => console.log('第二次');

// addEventListener 可以叠加，两个都会执行
function first() {
  console.log('第一次');
}
button.addEventListener('click', first);
button.addEventListener('click', () => console.log('第二次'));

// 移除时必须传入同一个函数引用，匿名函数无法移除
button.removeEventListener('click', first);
~~~

HTML 里写 `onclick="save()"` 也能用，但结构、行为、样式混在一起不好维护，推荐在脚本里用 `addEventListener` 绑定。普通函数作为处理函数时 `this` 指向绑定事件的元素，箭头函数没有自己的 `this`，会沿用外层作用域的值。

### 1.6 BOM 常用对象

`window` 表示浏览器窗口，`location` 表示当前地址，`history` 管理浏览历史，`localStorage` 保存本地字符串数据。

~~~javascript
console.log(location.href);
location.assign('/login');
localStorage.setItem('token', 'demo-token');
const token = localStorage.getItem('token');
~~~

不要把密码等高敏感信息直接放入 localStorage；令牌保存位置还要结合 XSS、CSRF 和项目认证方案评估。

#### 1.6.1 window、location、navigator 常用成员

`window` 是浏览器中最顶层的对象，也是全局作用域对象，`document`、`alert`、`localStorage` 和定时器都挂在它下面，所以调用时通常省略 `window.`。

| 成员 | 作用 |
|---|---|
| `window.innerWidth`、`window.innerHeight` | 视口宽高，用于响应式判断 |
| `window.alert`、`window.confirm`、`window.prompt` | 提示框、确认框、输入框 |
| `window.open`、`window.close` | 打开或关闭窗口 |
| `window.setTimeout`、`window.setInterval` | 定时器 |
| `window.localStorage`、`window.sessionStorage` | 本地存储，前者长期保存，后者关闭标签页即失效 |
| `window.addEventListener('resize', fn)` | 监听窗口尺寸变化 |

`location` 保存当前页面的地址信息，既能读取也能用它跳转：

| 成员 | 作用 |
|---|---|
| `location.href` | 完整地址，读取或赋值 |
| `location.protocol` | 协议，如 `https:` |
| `location.host` | 主机名和端口 |
| `location.pathname` | 路径部分 |
| `location.search` | `?` 开头的查询字符串 |
| `location.hash` | `#` 开头的锚点 |
| `location.assign(url)` | 跳转并保留历史记录 |
| `location.replace(url)` | 跳转但不保留历史记录 |
| `location.reload()` | 刷新当前页面 |

`navigator` 描述浏览器和运行环境：

| 成员 | 作用 |
|---|---|
| `navigator.userAgent` | 浏览器标识字符串，只适合做粗略统计，不要用来精确判断内核 |
| `navigator.language` | 浏览器语言，如 `zh-CN` |
| `navigator.onLine` | 当前是否联网 |
| `navigator.clipboard` | 读写剪贴板，需要用户授权或安全上下文 |

`history` 管理浏览历史，常用 `history.back()`、`history.forward()` 和 `history.go(-1)`；单页应用还会用 `history.pushState` 在不刷新页面的情况下修改地址。

~~~javascript
console.log(window.innerWidth, navigator.language, location.pathname);
window.addEventListener('resize', () => console.log('窗口尺寸变化'));
~~~

#### 1.6.2 定时器

`setTimeout` 表示等待一段时间后执行一次，`setInterval` 表示每隔一段时间重复执行。两者都返回一个数字 id，用 `clearTimeout`、`clearInterval` 取消。

| 方法 | 含义 | 取消方式 |
|---|---|---|
| `setTimeout(fn, ms)` | 延迟 `ms` 毫秒后执行一次 | `clearTimeout(id)` |
| `setInterval(fn, ms)` | 每隔 `ms` 毫秒执行一次 | `clearInterval(id)` |

~~~javascript
const timerId = setInterval(() => {
  console.log('每秒执行一次');
}, 1000);

// 五秒后停止轮询
setTimeout(() => {
  clearInterval(timerId);
}, 5000);
~~~

延时时间只是最短等待时间，不是精确时间：主线程被占用时回调会被推后。组件里创建的定时器必须在卸载阶段清理，否则组件销毁后回调仍然会执行，可能报错或造成内存泄漏。

## 2. Vue 基础

### 2.1 Vue 快速入门

Vue 是一个用于构建用户界面的渐进式 JavaScript 框架。它的核心思路是：用声明式模板描述界面，数据变化后由框架负责更新 DOM，开发者不需要自己写 `document.querySelector` 去改节点。

Vue 3 有两种常见引入方式：通过 CDN 引入后使用全局变量 `Vue`，或使用构建工具并通过 `import` 导入。使用 `import` 时脚本必须写成 `type="module"`，因为 ES Module 语法只在模块脚本里有效；同时页面要通过服务器访问（`file://` 直接打开会被浏览器的模块跨域策略拦截）。

~~~html
<script type="module">
  import { createApp } from 'vue';

  const app = createApp({
    data() {
      return { message: 'Hello Vue' };
    }
  });

  app.mount('#app');
</script>
~~~

挂载分三步：从 `vue` 导入 `createApp`，调用 `createApp({...})` 创建一个应用实例并传入根组件配置，最后调用 `.mount('选择器')` 把应用挂到页面元素上。上面的写法和后面章节使用的 `Vue.createApp({...}).mount('#app')` 是同一件事，只是导入方式不同。

挂载点选择器与模板的关系：

| 情况 | 说明 |
|---|---|
| 挂载点是空元素 | Vue 使用组件自身的 `template` 选项或 `.vue` 文件中的模板 |
| 挂载点内部有 HTML | 根组件没有 `template` 选项时，挂载点内部的 HTML 会被当作模板 |
| 选择器匹配不到元素 | 控制台出现警告，页面上不会有任何渲染结果 |
| 选择器匹配到多个元素 | 只使用第一个匹配到的元素作为挂载点，因此建议用唯一的 `id` |

挂载点内部的模板只能有一个根节点，`{{ }}` 里只能写表达式，不能写 `if`、`for` 这类语句。挂载完成后，还可以调用 `app.unmount()` 卸载应用。

### 2.2 MVVM 与双向绑定

MVVM 是 Model-View-ViewModel 的缩写，描述了 Vue 应用各部分的分工：

| 部分 | 在 Vue 中对应什么 | 职责 |
|---|---|---|
| Model | `data` 中的数据、从后端拿到的 JSON | 保存业务数据 |
| View | 页面上的 DOM，即模板渲染后的结果 | 展示数据、接收用户输入 |
| ViewModel | Vue 应用实例（`data`、`computed`、`methods` 等） | 在 Model 和 View 之间做同步 |

双向绑定指 ViewModel 同时负责两个方向的同步：数据变化时视图自动更新，用户修改表单控件时数据自动更新。前者由响应式系统完成，后者由 `v-model` 完成。

~~~html
<div id="app">
  <input v-model="name">
  <p>你好，{{ name }}</p>
</div>
~~~

这段代码里，输入框内容变化会写入 `name`，`name` 变化又会把 `<p>` 的文字更新，两个方向都不需要手写事件监听。注意 `v-model` 只对表单控件生效，普通文本插值 `{{ }}` 只是“数据到视图”的单向绑定。

### 2.3 响应式和模板

~~~html
<div id="app">
  <input v-model="keyword" placeholder="搜索部门">
  <ul>
    <li v-for="dept in filteredDepts" :key="dept.id">
      {{ dept.name }}
    </li>
  </ul>
</div>
<script>
Vue.createApp({
  data() {
    return {
      keyword: '',
      depts: [{ id: 1, name: '教研部' }]
    };
  },
  computed: {
    filteredDepts() {
      return this.depts.filter(d => d.name.includes(this.keyword));
    }
  }
}).mount('#app');
</script>
~~~

Vue 模板中的 `{{ }}` 是文本插值；表达式应保持简单，复杂计算放入 `computed`。修改响应式数据后，Vue 会在下一次更新周期刷新 DOM，可使用 `nextTick` 等待更新完成。

### 2.4 Vue 指令对比

| 指令 | 作用 | 示例 |
|---|---|---|
| v-bind 或冒号 | 绑定属性 | :disabled="loading" |
| v-model | 表单双向绑定 | v-model="form.name" |
| v-on 或 @ | 绑定事件 | @click="save" |
| v-if | 创建或销毁节点 | v-if="visible" |
| v-show | CSS 控制显示 | v-show="visible" |
| v-for | 列表渲染 | v-for="item in list" |

v-for 应使用稳定的 key，优先使用业务主键。

### 2.5 Vue 应用和组件

Vue 应用由根组件、数据、计算属性、方法和生命周期组成。计算属性适合根据已有数据计算新值，方法适合响应用户操作。

~~~javascript
const app = Vue.createApp({
  data() {
    return { count: 0 };
  },
  computed: {
    double() {
      return this.count * 2;
    }
  },
  methods: {
    increment() {
      this.count += 1;
    }
  }
});
app.mount('#app');
~~~

组件把页面拆成可复用的局部区域，父组件可以通过 props 向子组件传值，子组件可以通过事件通知父组件。

### 2.6 Vue 指令使用规范

| 指令 | 作用 | 注意事项 |
|---|---|---|
| `v-bind` / `:` | 绑定属性 | 动态属性不要写成普通字符串 |
| `v-model` | 表单双向绑定 | 适合 input、select、textarea |
| `v-on` / `@` | 绑定事件 | 事件方法写在 methods 中 |
| `v-if` | 条件创建或销毁 | 切换开销较大 |
| `v-show` | 使用 CSS 显示隐藏 | 频繁切换更合适 |
| `v-for` | 列表渲染 | 必须使用稳定的 `key` |
| `v-html` | 插入 HTML | 不要直接插入不可信内容 |

组件通信常见方式：父组件通过 `props` 传入数据，子组件通过自定义事件 `emit` 通知父组件；多个页面共享状态时再使用 Pinia 等状态管理工具。不要在子组件中直接修改 props。

#### 2.6.1 常用事件类型

`v-on` 或缩写 `@` 用来绑定事件，冒号后面是事件类型，等号后面是处理函数或表达式。`v-model` 负责的是数据绑定，`v-bind` 或 `:` 负责的是属性绑定，三者分工不要混。

| 事件类型 | 触发时机 | 典型用途 | 模板写法 |
|---|---|---|---|
| `click` | 鼠标点击元素 | 按钮、表格操作列 | `@click="save"`、`@click="save(item.id)"` |
| `input` | 输入内容每变化一次就触发 | 实时搜索、即时校验 | `@input="onInput"`，用 `event.target.value` 取值 |
| `change` | 值改变且失去焦点，或选择类控件选中后 | 下拉框、复选框、日期选择 | `@change="onChange"` |
| `submit` | 表单提交 | 查询、新增、登录表单 | `@submit.prevent="onSubmit"` |
| `blur` | 元素失去焦点 | 输入完成后校验 | `@blur="checkName"` |

~~~html
<input v-model="keyword" @input="search">
<select v-model="deptId" @change="loadEmps">
  <option value="1">教研部</option>
</select>
<form @submit.prevent="save">
  <button type="submit">保存</button>
</form>
~~~

`v-model` 内部已经处理了 `input` 或 `change` 事件以及 `value` 绑定，只是要同步数据时不需要再写 `@input`；需要额外做校验或发请求时再叠加事件。

常用修饰符：

| 修饰符 | 作用 |
|---|---|
| `.prevent` | 调用 `event.preventDefault()`，阻止表单默认提交或链接跳转 |
| `.stop` | 调用 `event.stopPropagation()`，阻止事件冒泡 |
| `.once` | 事件只触发一次 |
| `.trim` | 配合 `v-model`，自动去掉首尾空格 |

模板里不要写过长的表达式，处理逻辑写在 Options API 的 `methods` 中，组合式 API 中则写成 `setup` 里的函数。

## 3. Ajax 与 Axios

### 3.1 异步请求

同步代码会等待前一步执行完成，异步代码把任务交给浏览器，完成后通过回调、Promise 或 async/await 继续执行。网络请求应使用异步方式，避免阻塞页面。

#### 3.1.1 Ajax 是什么

Ajax 的全称是 Asynchronous JavaScript And XML（异步 JavaScript 和 XML）。它的作用是在不刷新整个页面的前提下，由 JavaScript 向服务器发送请求、拿到数据，再只更新页面的一部分。名字里的 XML 是历史叫法，现在前后端之间主要传输 JSON。

| 对比项 | 同步请求 | 异步请求（Ajax） |
|---|---|---|
| 页面表现 | 整页刷新或阻塞等待，可能出现白屏 | 页面不刷新，只更新局部 |
| 代码执行顺序 | 等待请求完成后才继续往下执行 | 请求发出后继续执行，完成后走回调 |
| 用户体验 | 操作被打断，等待时间长 | 可显示加载状态，交互流畅 |
| 典型场景 | 直接在地址栏打开接口 | 搜索联想、分页、表单提交 |

同步请求会让浏览器主线程停下来等待，同步的 `XMLHttpRequest` 已被标准废弃，实际项目中应始终使用异步请求。

#### 3.1.2 原生 XMLHttpRequest 的四个步骤

创建对象、打开连接、设置回调、发送请求：

| 步骤 | 代码 | 说明 |
|---|---|---|
| 1. 创建对象 | `new XMLHttpRequest()` | 不同浏览器历史上写法不同，现代浏览器统一使用 XMLHttpRequest |
| 2. 打开连接 | `xhr.open('GET', '/api/depts')` | 设置请求方式和地址；第三个参数为 `false` 表示同步，不要使用 |
| 3. 设置回调 | `xhr.onreadystatechange = fn` 或 `xhr.onload = fn` | 回调必须写在 `send()` 之前，否则可能错过触发时机 |
| 4. 发送请求 | `xhr.send()` | `POST` 时把请求体作为参数传入，如 `xhr.send(JSON.stringify(data))` |

原生 Ajax 的完整流程就是上面四步：创建 XMLHttpRequest 对象、调用 open 打开连接、设置回调、调用 send 发送请求：

~~~javascript
const xhr = new XMLHttpRequest();
xhr.open('GET', '/api/depts');
xhr.onload = () => {
  if (xhr.status >= 200 && xhr.status < 300) {
    console.log(JSON.parse(xhr.responseText));
  }
};
xhr.onerror = () => console.error('网络错误');
xhr.send();
~~~

为什么要判断 `readyState === 4 && status === 200`？`readyState` 表示请求走到哪一步，`status` 是服务器返回的 HTTP 状态码：

| readyState | 含义 |
|---|---|
| 0 | 未初始化，还没有调用 `open` |
| 1 | 已调用 `open`，连接已建立 |
| 2 | 已调用 `send`，收到响应头 |
| 3 | 正在接收响应体 |
| 4 | 请求完成，响应体全部收到 |

只有 `readyState === 4` 才代表响应体接收完毕，可以安全读取数据；只有状态码为 2xx 才代表服务器处理成功。两者都满足再去解析数据，可以避免读到半截响应，也能避免把 404、500 返回的错误页面当成正常数据。

`xhr.responseText` 是服务器返回的字符串，返回 JSON 时需要自己转换：

~~~javascript
xhr.onreadystatechange = () => {
  if (xhr.readyState === 4 && xhr.status === 200) {
    const result = JSON.parse(xhr.responseText);
    console.log(result.data);
  }
};

// onload 在 readyState 变为 4 时触发，可以少判断一次
xhr.onload = () => {
  if (xhr.status >= 200 && xhr.status < 300) {
    console.log(JSON.parse(xhr.responseText));
  }
};

// onreadystatechange 和 onload 都可以用，回调必须在 send 之前设置
xhr.onreadystatechange = () => {
  if (xhr.status !== 200) return;
  if (xhr.readyState === 4) {
    console.log(JSON.parse(xhr.responseText));
  }
};
~~~

原生写法比较啰嗦，判断状态码、转换 JSON、处理错误都要自己写，这也是实际项目改用 Axios 的主要原因。

~~~javascript
async function loadDepts() {
  try {
    const response = await fetch('/depts');
    // fetch 对 404/500 不一定自动抛异常
    if (!response.ok) throw new Error('HTTP ' + response.status);
    return await response.json();
  } catch (error) {
    console.error(error);
    return [];
  }
}
~~~

### 3.2 Axios 封装

~~~javascript
import axios from 'axios';

const http = axios.create({
  baseURL: '/api',
  timeout: 10000
});

async function queryDepts() {
  const { data } = await http.get('/depts');
  return data;
}

async function createDept(name) {
  await http.post('/depts', { name });
}
~~~

项目中建议统一配置 baseURL、超时、请求拦截器、响应拦截器和错误提示。

Axios 常用请求方法：

~~~javascript
http.get('/depts', { params: { name: '研发' } });
http.post('/depts', { name: '研发部' });
http.put('/depts/1', { name: '技术部' });
http.delete('/depts/1');
~~~

请求拦截器可统一添加 Token，响应拦截器可统一处理业务码和登录失效：

~~~javascript
http.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
~~~

前端校验用于即时反馈，但后端仍必须再次校验。使用 Fetch 时要检查 response.ok，因为 404/500 不一定让 Promise reject；长列表或页面切换时可用 AbortController 取消过期请求，避免旧响应覆盖新数据。跨域请求还会受到 CORS 策略限制。

请求失败应区分网络错误、HTTP 状态错误和业务错误：网络错误通常没有响应，HTTP 错误可通过 `error.response.status` 判断，业务错误则读取后端约定的 `code`。统一处理后再在页面显示用户能理解的消息。

### 3.3 Axios 的配置对象与响应结构

Axios 是基于 Promise 的请求库，内部封装了原生 XMLHttpRequest，会自动把 JSON 响应转换成对象，并支持拦截器。除了 `axios.get()` 这类别名方法，也可以只传一个配置对象：

~~~javascript
axios({
  method: 'post',
  url: '/depts',
  params: { page: 1, pageSize: 10 },
  data: { name: '教研部' },
  headers: { 'Content-Type': 'application/json' }
}).then(res => console.log(res.data));
~~~

配置对象的常用字段：

| 字段 | 作用 |
|---|---|
| `method` | 请求方式，如 `get`、`post`、`put`、`delete` |
| `url` | 请求地址，会与 `baseURL` 拼接 |
| `params` | 查询参数，拼到地址后面，如 `?page=1&pageSize=10` |
| `data` | 请求体数据，`post`、`put` 用它传 JSON |
| `headers` | 请求头，如 Token、`Content-Type` |
| `baseURL` | 统一前缀，配置在 `axios.create()` 中一次即可 |

请求别名方法和配置对象的对应关系：

| 别名写法 | 等价于 | 第二个参数是什么 |
|---|---|---|
| `axios.get(url, config)` | `axios({ method: 'get', url })` | 配置对象，查询参数放在 `config.params` |
| `axios.post(url, data, config)` | `axios({ method: 'post', url, data })` | 请求体数据 |
| `axios.put(url, data, config)` | `axios({ method: 'put', url, data })` | 请求体数据 |
| `axios.delete(url, config)` | `axios({ method: 'delete', url })` | 配置对象，查询参数放在 `config.params` |

`get` 请求没有请求体，所以 `axios.get(url, { params })` 中的对象是配置对象；`post`、`put` 的第二个参数是请求体，需要额外配置时要写到第三个参数。

~~~javascript
// 查询参数：请求地址是 /depts?name=研发
axios.get('/depts', { params: { name: '研发' } });

// 请求体：发送 {"name":"研发部"}
axios.post('/depts', { name: '研发部' });
~~~

无论成功还是失败，Axios 返回的响应对象结构一致：

| 属性 | 含义 |
|---|---|
| `res.data` | 服务器返回的数据，JSON 已经被自动转成对象 |
| `res.status` | HTTP 状态码，如 200 |
| `res.statusText` | 状态码对应的文本描述 |
| `res.headers` | 响应头 |
| `res.config` | 本次请求使用的配置 |

页面真正要用的业务数据在 `res.data` 里。如果后端统一返回 `{ code, msg, data }`，列表数据其实是 `res.data.data`，两层 `data` 容易看错，建议在响应拦截器里统一剥离一层再返回。

## 4. 前端工程化

### 4.1 组件和生命周期

生命周期是组件从创建、挂载、更新到卸载经历的一系列阶段，Vue 在每个阶段提供了钩子函数，让我们能在合适的时机执行代码。组件负责封装可复用的页面区域。

#### 4.1.1 八个生命周期钩子

Options API 的八个钩子按执行顺序如下，组合式 API 用 `on` 开头的函数在 `setup` 里注册同样的时机：

| 顺序 | Options API | 组合式 API | 执行时机 | 适合做什么 |
|---|---|---|---|---|
| 1 | `beforeCreate` | 没有独立钩子，`setup` 本身即此阶段 | 实例刚创建，数据和方法还没初始化 | 很少使用 |
| 2 | `created` | `setup` | 数据、方法已可用，DOM 还没生成 | 发起初始请求、加工初始数据 |
| 3 | `beforeMount` | `onBeforeMount` | 模板编译完成，尚未挂载到页面 | 很少使用 |
| 4 | `mounted` | `onMounted` | DOM 已挂载，可以访问 `$el` | 操作 DOM、初始化图表、开启定时器 |
| 5 | `beforeUpdate` | `onBeforeUpdate` | 数据已变化，DOM 还没更新 | 记录更新前的 DOM 状态 |
| 6 | `updated` | `onUpdated` | DOM 已按新数据更新完成 | 依赖最新 DOM 的收尾处理，不要在这里改数据 |
| 7 | `beforeUnmount` | `onBeforeUnmount` | 组件即将卸载，DOM 仍然存在 | 取消定时器、解绑监听、取消未完成请求 |
| 8 | `unmounted` | `onUnmounted` | 组件已卸载，DOM 被移除 | 释放全局资源、销毁第三方实例 |

Vue 3 中 `beforeDestroy` 和 `destroyed` 已改名为 `beforeUnmount` 和 `unmounted`。写旧名字不会报错，钩子只是不会执行，从 Vue 2 迁移代码时要特别注意。

~~~mermaid
flowchart TD
    A[beforeCreate] --> B[created]
    B --> C[beforeMount]
    C --> D[mounted]
    D --> E{数据发生变化?}
    E -- 是 --> F[beforeUpdate]
    F --> G[updated]
    G --> E
    E -- 组件被卸载 --> H[beforeUnmount]
    H --> I[unmounted]
~~~

#### 4.1.2 两种 API 的钩子写法

同一次开发只用一种写法。Options API 的钩子写在组件配置对象里，与 `data`、`methods` 平级，下面这段只使用 Options API 钩子：

~~~javascript
Vue.createApp({
  data() {
    return { timer: null };
  },
  created() {
    // 数据已可用，适合发起初始请求
    console.log('created');
  },
  mounted() {
    // DOM 已挂载，适合初始化定时器
    this.timer = setInterval(() => console.log('tick'), 1000);
  },
  beforeUnmount() {
    // 组件卸载前清理，避免回调继续执行
    clearInterval(this.timer);
  }
}).mount('#app');
~~~


组合式 API 在 `setup` 中注册钩子，下面的写法里所有钩子都来自组合式 API，不要和 Options API 的 `created`、`beforeUnmount` 混在同一段代码中：

~~~javascript
import { createApp, ref, onMounted, onUnmounted } from 'vue';

createApp({
  setup() {
    const timer = ref(null);

    // setup 本身相当于 created 阶段，适合准备数据
    onMounted(() => {
      timer.value = setInterval(() => console.log('tick'), 1000);
    });

    onUnmounted(() => {
      clearInterval(timer.value);
    });

    return { timer };
  }
}).mount('#app');
~~~

`onMounted`、`onUnmounted` 等钩子必须在 `setup` 中同步调用，放在异步回调或条件语句里可能不会注册成功。要不要清理资源，判断标准是“这个副作用是否会在组件销毁后继续运行”，定时器、事件监听、WebSocket、轮询都会。

### 4.2 Vue Router 和 Element Plus

Vue Router 管理页面路由；Element Plus 提供 Table、Pagination、Dialog、Form 等组件。

Vue 项目常见开发流程是：创建项目、安装依赖、划分组件、定义路由、封装请求、联调接口、执行构建。生产构建通常使用 `npm run build`，生成的静态文件再部署到 Web 服务器。

生命周期常见顺序为：创建组件、挂载 DOM、更新数据、卸载组件。定时器、事件监听和 WebSocket 应在卸载阶段清理，避免组件销毁后仍然执行回调。

### 4.3 Element Plus 常用组件

| 组件 | 用途 | 关键属性或事件 |
|---|---|---|
| Table | 展示列表 | `data`、`prop`、`label` |
| Pagination | 分页 | `current-page`、`page-size`、`current-change` |
| Dialog | 弹窗 | `v-model`、`title`、`width` |
| Form | 表单校验 | `model`、`rules`、`ref` |

列表页通常由查询表单、表格、分页和新增/编辑弹窗组成。分页组件变化时重新请求后端，删除成功后刷新当前页；表单提交前调用校验方法，失败时定位到具体字段。

### 4.4 路由与前端部署

路由把 URL 映射到页面组件。前端路由切换时应显示加载状态，登录后才能访问的页面要做路由守卫，但真正的权限校验仍由后端完成。

生产构建命令通常是 `npm run build`，构建结果位于 `dist`。部署时把 `dist` 放到 Nginx 等 Web 服务器，并配置历史路由回退到 `index.html`；API 代理或 CORS 要与后端地址保持一致。

## 5. 小白易错点

- `<script type="module">` 里使用 `import`，却用 `file://` 双击打开页面，浏览器会拦截模块加载，必须通过服务器访问。
- 挂载点选择器写错（`#app` 写成 `app`），页面空白，控制台提示找不到元素。
- 调用了 `createApp` 却忘记 `.mount()`，应用永远不会渲染。
- 在 Vue 3 中使用 `beforeDestroy`、`destroyed`，钩子已改名为 `beforeUnmount`、`unmounted`，旧名字不会执行也不报错。
- Options API 的 `created`、`beforeUnmount` 与组合式 API 的 `setup`、`onMounted` 混在同一段代码里，容易取不到数据或钩子不生效。
- 子组件直接修改 `props`，Vue 会告警，数据流也变得难以追踪。
- `axios.get('/depts', { name: '研发' })` 想让 `name` 变成查询参数，实际必须写成 `{ params: { name: '研发' } }`。
- `axios.post(url, data)` 的第二个参数是请求体数据，配置对象要放到第三个参数，写错位置会导致参数发不出去。
- 把 Axios 的响应对象当成数据本身使用，忘记取 `res.data`。
- 使用原生 `XMLHttpRequest` 时不判断 `readyState` 和 `status` 就 `JSON.parse`，拿到 404 页面时直接抛错。
- 用 `open` 的第三个参数传 `false` 发同步请求，页面被阻塞，该方法已被废弃。
- 组件里创建的定时器、监听器没有在卸载阶段清理，组件销毁后回调继续执行。
- `setTimeout` 的延时时间是最短等待时间，用它做精确定时不可靠。

## 6. 练习清单

1. 用 `<script type="module">` 加 `import { createApp } from 'vue'` 写一个挂载到 `#app` 的最小应用，并验证选择器写错时的控制台提示。
2. 用 `v-model` 做一个输入框实时显示字数，再用 `computed` 显示字数是否超限。
3. 用 `v-bind` 和 `v-on` 实现一个按钮：输入为空时禁用，点击后计数加一。
4. 对同一个按钮分别使用 `onclick` 和 `addEventListener` 绑定两个处理函数，验证覆盖与叠加的差别，并移除其中一个。
5. 用原生 `XMLHttpRequest` 请求一个返回 JSON 的接口，打印 `responseText` 和解析后的对象，再故意请求一个不存在的地址观察状态码。
6. 分别用配置对象和别名方法各发一次 GET、POST 请求，对比 `res.data` 与 `res.status` 的内容。
7. 在组件中开启一个 `setInterval`，用 `beforeUnmount` 和 `onUnmounted` 两种写法各清理一次，确认定时器停止。

## 7. 资料对应关系

- 《JavaWeb笔记》：第二章“前端 Web 开发（JavaScript + Vue + Ajax）”。
- Vue 3 官方文档：[简介](https://cn.vuejs.org/guide/introduction.html)。
- MDN 参考：[XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)。
- Axios 官方文档：[Axios 起步](https://axios-http.com/zh/docs/intro)。
- 上一章 [01-前端Web开发（HTML+CSS）.md](01-前端Web开发（HTML+CSS）.md) 提供了本节用到的页面结构、路径与样式基础。

## 8. 本章总结

- JavaScript 负责行为，DOM 负责文档操作。
- Vue 使用响应式数据驱动视图，Axios 负责 HTTP 请求。
- 组件、路由和组件库是前端工程化基础。
