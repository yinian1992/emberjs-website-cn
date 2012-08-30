## 简介

### Ember.js 是什么？

Ember 是一个用于创建野心勃勃的 web 应用程序的 JavaScript 框架，它免除了
样板文件并提供了一套标准的应用程序架构。

#### 消灭样板

对每个 web 应用程序都有一些任务是共同的。例如，从服务器获取数据，把它渲染到
屏幕上，然后在数据变更的时候更新结果。

因为浏览器提供的做这件事情的工具很原始，你到头来会一遍又一遍编写相同的代码。
Ember.js 提供的工具让你专注于你的应用，而不是去写你已经写过八百遍的相同的
代码。

因为我们自己已经构建了许多应用，我们已经超越了显而易见的低级事件驱动抽象，免
除了大部分在整个应用中，尤其是向 DOM 本身中传播变更相关的样板文件。

要完成在视图中管理变更，Ember.js 提供了一个会在底层对象变更时自动更新 DOM
的模板引擎。

比如这个简单的例子，一个 Person 的模板:

```handlebars
{{person.name}} is {{person.age}}.
```

如同任何模板系统，当模板第一次渲染后，它会反射当前的 person 的状态。为了避
免样板文件，Ember.js 也会在 person 的 name 或 age 变更时自动为你更新 DOM。

一旦你确定了样板，Ember.js 就会保证它总是最新的。

#### 提供架构

既然 web 应用从只不过是静态文档的网页进化而来，浏览器提供的东西少到令人沮丧。

Ember 使得把应用划分为模型、视图和控制器十分简单，这样不仅改善了可测试性，模
块化代码，也方便项目中的新开发者快速理解一切是怎么配合在一起的。那些充斥着裹
脚布一般的回调的日子一去不返。

Ember 同样提供了状态管理的内置支持，所以你会发现创造性的方式来描述你的应用如
何在各种嵌套状态（譬如已登出、已登入、文章浏览中和评论浏览中）中转移。

### Ember.js 如何与众不同？

传统的 web 应用会在用户每次与服务器交互时下载一个新页面。这意味着每次交互都不
会比你和用户之间的延迟快，而且通常更慢。在页面中的一部分使用 AJAX 会有所帮助，
但每次你的 UI 需要更新时仍然需要一次到服务器的往返。并且如果页面中的多个部分
需要一次性全部更新，大多数开发人员只是靠重新加载页面，因为保证整个页面同步是
很棘手的。

Ember.js，像一些其它的现代 JavaScript 框架一样，而实现略有不同。一个 Ember.js
应用在第一次页面加载时就下载了所以它需要运行的东西，而不是把大多数的应用逻辑
存放在服务器上。这意味着当你的用户使用你的应用时，她不需要加载一个新页面，这
样 UI 会快速响应他们的交互。

这个架构的一个优势是你的 web 应用使用与你本地应用或第三方客户端相同的 REST
 API。后端开发者也可以专心构建一个快速、可靠且安全的 API 服务器，而不再需要
成为一个前端专家。

### 初瞥 Ember.js

有三个特性让使用 Ember 成为乐趣：

1. 绑定
2. 计算属性
3. 自动更新的模板

#### 绑定

绑定用于保持两个不同对象的属性同步。你只需要声明绑定一次，Ember 就会保证变
更会双向传播。

如此，你可以创建两个对象间的一个绑定：

```javascript
MyApp.president = Ember.Object.create({
  name: "Barack Obama"
});

MyApp.country = Ember.Object.create({
  // 以“Binding”结尾的属性名会让 Ember 创建
  // 一个到 presidentName 属性的绑定。
  presidentNameBinding: 'MyApp.president.name'
});

// 随后，在 Ember 已经解析绑定后……
MyApp.country.get('presidentName');
// "Barack Obama"
```

绑定允许你以 MVC 模式构建你的应用程序，之后剩下的部分易于获知数据始终会
在层与层之间正确地流动。

#### 计算属性

计算属性允许你像对待一个属性一样处理一个函数：

```javascript
MyApp.president = Ember.Object.create({
  firstName: "Barack",
  lastName: "Obama",

  fullName: function() {
    return this.get('firstName') + ' ' + this.get('lastName');

  // 调用这个标志来把一个函数标记为属性
  }.property()
});

MyApp.president.get('fullName');
// "Barack Obama"
```

计算属性是有用的，因为它可以像其它属性一样与绑定协同工作。

许多计算属性会依赖于其它属性。例如，在上例中，`fullName` 属性依赖
`firstName` 和 `lastName` 来决定它的值。你可以像这样告知 Ember 这
些依赖：

```javascript
MyApp.president = Ember.Object.create({
  firstName: "Barack",
  lastName: "Obama",

  fullName: function() {
    return this.get('firstName') + ' ' + this.get('lastName');

  // 告知 Ember 这个计算属性依赖于 firstName
  // 和 lastName
  }.property('firstName', 'lastName')
});
```

确保你列出这些以来，这样 Ember 才知道何时更新一个连接到一个计算
属性的依赖。

#### 自动更新的模板

Ember 使用了 Handlebars，一个语义化的模板库。要从你的 JavaScript
应用中取出数据并把它放进 DOM 里，在 HTML 中任意你想要值出现的地方放
置一个 `<script>` 标签：

```handlebars
<script type="text/x-handlebars">
  The President of the United States is {{MyApp.president.fullName}}.
</script>
```

这是最棒的部分：模板是绑定可知的。也即是如果你更改了你要模板显示的值，
我们会自动为你更新。并且，因为你已经指定了依赖， *那些* 属性的更改也
会反射。

希望你能了解这三个强大的工具是如果在一起工作的：从一些原始属性开始，
然后开始用计算属性构建更复杂的属性和依赖。一旦你已经描述了数据，你只
需指定一次它如何展示，Ember 会完成剩下的。无论你底层数据如何变更，或
是从 XHR 请求或是用户执行一个动作；你的用户界面始终会保持最新。这消
除了开发人员每天艰苦斗争的所有类型的边缘状况。
