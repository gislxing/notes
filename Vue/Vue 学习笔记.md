# Vue 学习笔记

## 安装

### 直接用`<script>`引入

直接下载并用 `<script>` 标签引入，`Vue` 会被注册为一个全局变量。

[开发版](https://vuejs.org/js/vue.js)

[生产版](https://vuejs.org/js/vue.min.js)

### CDN

对于制作原型或学习，你可以这样使用最新版本：

```html
<script src="https://cdn.jsdelivr.net/npm/vue"></script>
```

对于生产环境，我们推荐链接到一个明确的版本号和构建文件，以避免新版本造成的不可预期的破坏：

```html
<script src="https://cdn.jsdelivr.net/npm/vue@2.6.10/dist/vue.js"></script>
```

如果你使用原生 ES Modules，这里也有一个兼容 ES Module 的构建文件：

```html
<script type="module">
  import Vue from 'https://cdn.jsdelivr.net/npm/vue@2.6.10/dist/vue.esm.browser.js'
</script>
```

请确认了解[不同构建版本](https://cn.vuejs.org/v2/guide/installation.html#%E5%AF%B9%E4%B8%8D%E5%90%8C%E6%9E%84%E5%BB%BA%E7%89%88%E6%9C%AC%E7%9A%84%E8%A7%A3%E9%87%8A)并在你发布的站点中使用**生产环境版本**，把 `vue.js` 换成 `vue.min.js`。这是一个更小的构建，可以带来比开发环境下更快的速度体验。

### 命令行工具 (CLI)

## 指令

指令 (Directives) 是带有 `v-` 前缀的特殊特性。指令的值预期是**单个 JavaScript 表达式** (`v-for`例外)。指令的职责是，当表达式的值改变时，将其产生的连带影响，响应式地作用于 DOM

```html
<!-- 引入vue.js文件 -->
<script src="./vue.js"></script>

<!-- 挂载点 -->
<div id="app">{{content}}</div>

<script>
  new Vue({
    el: "#app",	// vue处理的范围，即id=app节点下所有内容都有vue渲染
    data: {			// 定义数据
      content: "hello world"
    }
  });
  
  // 2秒之后变更content内容
  setTimeout(function(){
    app.content = "HELLO WORLD";
    // app.$data.content = "HELLO WORLD";
  }, 2000);
</script>
```

可以用方括号括起来的 `JavaScript` 表达式作为一个指令的参数：

```html
<a v-bind:[attributeName]="url"> ... </a>
```

这里的 `attributeName` 会被作为一个 JavaScript 表达式进行动态求值，求得的值将会作为最终的参数来使用。例如，如果你的 Vue 实例有一个 `data` 属性 `attributeName`，其值为 `"href"`，那么这个绑定将等价于 `v-bind:href`。

### v-text

更新元素的 `textContent`，内容以字符串的形式插入

```html
<span v-text="msg"></span>
```

### v-html

更新元素的 `innerHTML`，内容按普通 HTML 插入

```html
<span v-html="html"></span>
```

### v-if 条件指令

```html
// 控制切换一个元素是否显示：
<div id="app-3">
  <p v-if="seen">现在你看到我了</p>
</div>
```

```js
var app3 = new Vue({
  el: '#app-3',
  data: {
    // seen = false，v-if 则删除 p 元素
    seen: true
  }
})
```

### v-else

不需要表达式，前一兄弟元素必须有 `v-if` 或 `v-else-if`

`v-else` 元素必须紧跟在带 `v-if` 或者 `v-else-if` 的元素的后面，否则它将不会被识别

```html
<div v-if="Math.random() > 0.5">
  Now you see me
</div>
<div v-else>
  Now you don't
</div>
```

### v-else-if

前一兄弟元素必须有 `v-if` 或 `v-else-if`

表示 `v-if` 的 “else if 块”。可以链式调用。

```html
<div v-if="type === 'A'">
  A
</div>
<div v-else-if="type === 'B'">
  B
</div>
<div v-else-if="type === 'C'">
  C
</div>
<div v-else>
  Not A/B/C
</div>
```

### v-show 条件指令

```html
// 控制切换一个元素是否显示：
<div id="app-3">
  <p v-show="seen">现在你看到我了</p>
</div>
```

```js
var app3 = new Vue({
  el: '#app-3',
  data: {
    // seen = false，v-show 则隐藏 p 元素
    seen: true
  }
})
```

###  v-for 循环

`v-for` 指令可以绑定数组的数据来渲染一个项目列表：

```html
<ul>
  <!-- list 是Vue的data里面的list，循环会将list中的每一项放到item中-->
	<li v-for="item in list">{{item}}</li>
</ul>
```

### v-on 事件监听器

`v-on` 指令添加一个事件监听器，通过它调用在 Vue 实例中定义的方法

简写：@

```html
<div id="app">
  <input type="text">
  <button v-on:click="handleSubmit">提交</button>
  <!-- v-on 简写成 @ -->
  <button @click="handleSubmit">提交</button>
  <ul>
    <li v-for="item in list">{{item}}</li>
  </ul>
</div>

<script>
  new Vue({
    el: '#app',
    data: {
      list: []
    },
    methods: {
      handleSubmit() {
        alert(333);
      }
    }
  });
</script>
```

### v-model 双向数据绑定

```html
<!-- input输入框的内容发生变化，那么Vue里的data.message也会自动跟着变化 -->
<div id="app">
  <p>{{ message }}</p>
  <input v-model="message">
</div>

<script>
  var app = new Vue({
    el: '#app',
    data: {
      message: 'Hello Vue!'
    }
  });
</script>
```

### v-bind 属性绑定

将元素节点的属性和 Vue 实例的属性保持一致

简写：冒号（:）

```html
<div id="app-2">
  <span v-bind:title="message">
    <!-- 简写 <span :title="message"> -->
    鼠标悬停几秒钟查看此处动态绑定的提示信息！
  </span>
</div>
```

```js
var app2 = new Vue({
  el: '#app-2',
  data: {
    message: '页面加载于 ' + new Date().toLocaleString()
  }
})
```

这里将 `span` 的 `title` 属性的值和 `Vue` 的`message` 保持一致

## Vue 实例

### 创建 Vue 实例

每个 Vue 应用都是通过用 `Vue` 函数创建一个新的 **Vue 实例**开始

```js
var vm = new Vue({
  // 选项
})
```

一个 Vue 应用由一个通过 `new Vue` 创建的**根 Vue 实例**，以及可选的嵌套的、可复用的组件树组成。

### 数据与方法

当一个 Vue 实例被创建时，它将 `data` 对象中的所有的属性加入到 Vue 的**响应式系统**中。当这些属性的值发生改变时，视图将会产生“响应”，即匹配更新为新的值。

只有当 Vue 实例被创建时 `data` 中存在的属性才是**响应式**的。也就是说如果你添加一个新的属性，那么属性的改变不会触发任何视图的更新

`Object.freeze()`会阻止修改现有的属性，也意味着响应系统无法再*追踪*变化

```js
var obj = {
  foo: 'bar'
}

Object.freeze(obj)

new Vue({
  el: '#app',
  data: obj
})
```

```html
<div id="app">
  <p>{{ foo }}</p>
  <!-- 这里的 `foo` 不会更新！ -->
  <button v-on:click="foo = 'baz'">Change it</button>
</div>
```

Vue 实例还暴露了一些有用的实例属性与方法。它们都有前缀 `$`

### 实例生命周期钩子

每个 Vue 实例在创建、销毁、数据变化、渲染等生命周期过程中会运行一些叫做**生命周期钩子**的函数

生命周期钩子的 `this` 上下文指向调用它的 Vue 实例

```js
new Vue({
  data: {
    a: 1
  },
  created: function () {
    // this 指向 vm 实例
    console.log('a is: ' + this.a)
  }
})
```

不要在**选项属性**或**回调**上使用**箭头函数**，比如 `created: () => console.log(this.a)` 或 `vm.$watch('a', newValue => this.myMethod())`。因为箭头函数并没有 `this`

## 组件

在 Vue 里，一个组件本质上是一个拥有预定义选项的一个 Vue 实例

**如果定义的组件名称是驼峰命令，那么在dom中使用的时候，需要将_大写字母_换成 _-小写字母_**

例如：myComponent  在HTML中就要写成：<my-component></my-component>

```js
// 定义名为 todo-item 的新组件(全局)
Vue.component('todo-item', {
  template: '<li>这是个待办项</li>'
})

// 局部组件(局部组件必须注册)
let todoItem = {
  props: ['content'],								// 接收传入子组件的参数
  template: '<li>{{content}}</li>'	// 组件的模板
}

// 注册局部组件
new Vue({
  el: '#app',
  components: {
    todoItem
  }
});
```

创建好组件后就可直接使用

```html
<!-- 创建一个 todo-item 组件的实例 -->
<todo-item></todo-item>
```

### 父组件给子组件传值

```html
<!-- 将父组件的值item绑定到自定义属性content上 -->
<todo-item 
    v-bind:content="item" 
    v-for="item in list">
</todo-item>
```

```js
// 子组件通过在 props 属性接收 content 的值
Vue.component('todo-item', {
  props: ['content'],
  template: '<li>{{content}}</li>'
});
```

### 子组件给父组件传值

```js
methods: {
  itemClick: function() {
    // 向父组件传递一个自定义事件，父组件可使用 v-on 监听该事件
    // 第一个参数：自定义事件名称
    // 第2...参数：作为自定义事件的参数
    this.$emit('del', this.index);
  }
}
```

```html
// 监听子组件传递的事件
<自定义组件 v-on:del="delItem"> </自定义组件>
```

```js
// 在父组件中定义处理该事件的方法
new Vue({
  ...
  ...
  methods: {
    delItem: function(index) {
      // do somthing
    }
  }
});
```

## 模板语法

### 插值

数据绑定最常见的形式就是使用`Mustache`语法 (双大括号) 的文本插值：

```html
<span>Message: {{ msg }}</span>
```

`Mustache` 标签将会被替代为对应数据对象上 `msg` 属性的值。无论何时，绑定的数据对象上 `msg`属性发生了改变，插值处的内容都会更新。

### 使用 JavaScript 表达式

对于所有的数据绑定，`Vue` 都支持完全的` JavaScript` 表达式

每个绑定都只能包含**单个表达式**

```html
{{ number + 1 }}

{{ ok ? 'YES' : 'NO' }}

{{ message.split('').reverse().join('') }}

<div v-bind:id="'list-' + id"></div>

<!-- 这是语句，不是表达式 -->
{{ var a = 1 }}
```

## 计算属性和侦听器

### 计算属性

对于复杂逻辑应当使用**计算属性**

可以像绑定普通属性一样在模板中绑定计算属性

```html
<div id="example">
  <p>Original message: "{{ message }}"</p>
  <p>Computed reversed message: "{{ reversedMessage }}"</p>
</div>
```

```js
var vm = new Vue({
  el: '#example',
  data: {
    message: 'Hello'
  },
  computed: {
    // 计算属性的 getter
    reversedMessage: function () {
      // `this` 指向 vm 实例
      return this.message.split('').reverse().join('')
    }
  }
})
```

**计算属性是基于它们的响应式依赖进行缓存的**，只在相关响应式依赖发生改变时它们才会重新求值。只要 `message` 还没有发生改变，多次访问 `reversedMessage`计算属性会立即返回之前的计算结果，而不必再次执行函数。

```js
computed: {
  now: function () {
    // 因为 Date.now() 不是响应式依赖, 所以此处的计算属性将不会更新
    return Date.now()
  }
}
```

### 计算属性的 setter

计算属性默认只有 getter ，不过在需要时你也可以提供一个 setter ：

```js
computed: {
  fullName: {
    // getter
    get: function () {
      return this.firstName + ' ' + this.lastName
    },
    // setter
    set: function (newValue) {
      var names = newValue.split(' ')
      this.firstName = names[0]
      this.lastName = names[names.length - 1]
    }
  }
}
```

### 侦听器 watch

数据变化时执行异步或开销较大的操作时，这个方式是最有用的

```js
var watchExampleVM = new Vue({
  el: '#watch-example',
  data: {
    question: ''
  },
  watch: {
    // 如果 `question` 发生改变，这个函数就会运行
    question: function (newQuestion, oldQuestion) {
      // do somting ...
    }
  }
});
```

## Class 与 Style 绑定

用 `v-bind` 绑定属性：只需要通过表达式计算出字符串结果即可

在将 `v-bind` 用于 `class` 和 `style` 时，表达式结果的类型除了字符串之外，还可以是对象或数组。

### 绑定 HTML Class

我们可以传给 `v-bind:class` 一个**对象**，以动态地切换 class：

```html
<!-- active 这个 class 存在与否将取决于数据属性 isActive 的值（true / false）。 -->
<div v-bind:class="{ active: isActive, 'text-danger': isDanger }"></div>
```

绑定的数据对象不必内联定义在模板里：

```html
<div v-bind:class="classObject"></div>
```

```js
data: {
  classObject: {
    active: true,
    'text-danger': false
  }
}
```

把一个**数组**传给 `v-bind:class`，以应用一个 class 列表：

```html
<div v-bind:class="[activeClass, errorClass]"></div>
```

```js
data: {
  activeClass: 'active',
  errorClass: 'text-danger'
}
```

渲染为：

```html
<div class="active text-danger"></div>
```

在数组语法中也可以使用对象语法：

```html
<div v-bind:class="[{ active: isActive }, errorClass]"></div>
```

### 绑定内联样式

#### 对象语法

`v-bind:style` 实是一个 JavaScript 对象。CSS 属性名可以用驼峰式 (camelCase) 或短横线分隔 (kebab-case，记得用引号括起来) 来命名：

```html
<div v-bind:style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>
```

```js
data: {
  activeColor: 'red',
  fontSize: 30
}
```

也可以直接绑定对象：

```html
<div v-bind:style="styleObject"></div>
```

```js
data: {
  styleObject: {
    color: 'red',
    fontSize: '13px'
  }
}
```

#### 数组语法

`v-bind:style` 的数组语法可以将多个样式对象应用到同一个元素上：

```html
<div v-bind:style="[baseStyles, overridingStyles]"></div>
```

## 条件渲染

### 用 `key` 管理可复用的元素

Vue 会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address">
</template>
```

那么在上面的代码中切换 `loginType` 将不会清除用户已经输入的内容。因为两个模板使用了相同的元素，`<input>` 不会被替换掉——仅仅是替换了它的 `placeholder`

需要替换掉原有的元素，则添加一个具有唯一值的 `key` 属性即可，`key`保证多个相同的元素完全独立

```html
<template v-if="loginType === 'username'">
  <label>Username</label>
  <input placeholder="Enter your username" key="username-input">
</template>
<template v-else>
  <label>Email</label>
  <input placeholder="Enter your email address" key="email-input">
</template>
```

## 列表渲染

建议尽可能在使用 `v-for` 时提供 `key` attribute

### 用 `v-for` 把数组对应为一组元素

我们可以用 `v-for` 指令把一个数组渲染为一个列表。`v-for` 指令需要使用 `item in items` 形式的特殊语法，其中 `items` 是源数据数组，而 `item` 则是被迭代的数组元素的**别名**。

```html
<ul id="example-1">
  <li v-for="item in items">
    {{ item.message }}
  </li>
</ul>
```

```js
var example1 = new Vue({
  el: '#example-1',
  data: {
    items: [
      { message: 'Foo' },
      { message: 'Bar' }
    ]
  }
})
```

`v-for` 还支持可选的第二个参数，即当前项的索引

```html
<ul id="example-2">
  <li v-for="(item, index) in items">
    {{ index }} - {{ item.message }}
  </li>
</ul>
```

你也可以用 `of` 替代 `in` 作为分隔符

```html
<div v-for="item of items"></div>
```

### 在 `v-for` 里使用对象

你也可以用 `v-for` 来遍历一个对象的属性。

```html
<ul id="v-for-object" class="demo">
  <li v-for="value in object">
    {{ value }}
  </li>
</ul>
```

```js
new Vue({
  el: '#v-for-object',
  data: {
    object: {
      title: 'How to do lists in Vue',
      author: 'Jane Doe',
      publishedAt: '2016-04-10'
    }
  }
})
```

第二个参数为 property 名称 (也就是键名)：

```html
<div v-for="(value, name) in object">
  {{ name }}: {{ value }}
</div>
```

第三个参数为索引：

```html
<div v-for="(value, name, index) in object">
  {{ index }}. {{ name }}: {{ value }}
</div>
```

### 数组更新检测

直接利用索引直接设置一个数组项时，Vue 检测不到数组变动，此时需要使用 `vm.$set` 实例方法或者`Vue.set`全局方法

#### 变异方法 (mutation method)

以下变异方法将会触发视图更新：

- `push()`
- `pop()`
- `shift()`
- `unshift()`
- `splice()`
- `sort()`
- `reverse()`

#### 替换数组

**变异方法**会改变原始数组。非变异 (non-mutating method) 方法，例如 `filter()`、`concat()` 和 `slice()` 。它们不会改变原始数组，而**是返回一个新数组**。当使用非变异方法时，可以用新数组替换旧数组：

```js
example1.items = example1.items.filter(function (item) {
  return item.message.match(/Foo/)
})
```

### 对象变更检测注意事项

**Vue 不能检测对象属性的添加或删除**

对于已经创建的实例，Vue 不允许动态添加根级别的响应式属性。但是，可以使用 `Vue.set(object, propertyName, value)` 方法向嵌套对象添加响应式属性

```js
var vm = new Vue({
  data: {
    userProfile: {
      name: 'Anika'
    }
  }
})
```

添加一个新的 `age` 属性到嵌套的 `userProfile` 对象：

```js
Vue.set(vm.userProfile, 'age', 27)
```

### Template 上使用 v-for

`template` 元素是 `Vue` 的标记元素，它不会渲染到页面

```html
<ul>
  <template v-for="item in items">
    <li>{{ item.msg }}</li>
    <li class="divider" role="presentation"></li>
  </template>
</ul>
```

