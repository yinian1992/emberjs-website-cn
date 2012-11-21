# Ember.js 的视图层

本指导会详尽阐述 Ember.js 视图层的细节。为想成为熟练 Ember 开发者准备，且包
含了对于入门 Ember 不必要的细节。

Ember.js 有一套复杂的用于创建、管理并渲染连接到浏览器 DOM 上的层级视图的系
统。视图负责响应诸如点击、拖拽以及滚动等的用户事件，也在视图底层数据变更时更
新 DOM 的内容。

视图层级通常由求值一个 Handlebars 模板创建。当模板求值后，会添加子视图。当
_那些_ 子视图求值后，会添加它们的子视图，如此递推，直到整个层级被创建。

即使你并没有在 Handlebars 模板中显式地创建子视图，Ember.js 内部仍使用视图系
统更新绑定的值。例如，每个 Handlebars 表达式 `{{value}}` 幕后创建一个视图，
这个视图知道当值变更时如何更新绑定值。

你也可以在应用运行时用 `Ember.ContainerView` 类对视图层级做出修改。一个容器
视图暴露一个可以手动修改的子视图实例数组，而非模板驱动。

视图和模板串联工作提供一套用于创建任何你梦寐以求的用户界面的稳健系统。最终用
户应从诸如当渲染和事件传播是的计时事件之类的复杂东西中隔离开。应用开发者应可
以一次性把他们的 UI 描述成 Handlebars 标记字符串，然后继续完成他们的应用，而
不必烦恼于确保它一直是最新的。

## 它解决了什么问题？

### 子视图

在典型的客户端应用中，视图同时在本身和 DOM 中表示嵌套的元素。在解决这个问题
的天真方案中，独立的视图对象表示单个 DOM 元素，专门的引用解决不同种类的视图
保持对概念中嵌套在它们内部的视图的跟踪。


这里是一个简单的例子，表示一个应用主视图，里面有一集合嵌套视图，且独立的元素在集合内嵌套。

<figure>
  <img src="/images/view-guide/view-hierarchy-simple.png">
</figure>

这个系统第一眼看上去毫无异样，但是想象我们要在上午 8 点而不是上午 9 点开放乔
的七鳃鳗小屋。在这种情况下，我们会想要重新渲染应用视图。因为开发者需要构建指
向在一个特殊基础上的子视图的引用，这个重渲染过程存在若干问题。

为了重新渲染应用视图，应用视图也必须手动重新渲染子视图并重新把它们插入到应用
视图的元素中。如果实现得完美，这个过程会正常工作，但它依赖于一个完美的，专门
的视图层级实现。如果任何一个视图没有精确地实现它，整个重新渲染过程会失败。

为了避免这些问题，Ember 的视图层级从概念上就带有子视图的烙印。

<figure>
  <img src="/images/view-guide/view-hierarchy-ember.png">
</figure>

当应用时图重新渲染时，Ember 而不是应用代码负责重新渲染并插入子视图。这也意味
着 Ember 可以为你执行任何内存管理，比如清理观察者和绑定。

这不仅在一定程度上消灭了样板代码，也破除了有瑕疵的视图层级实现带来的未期失败
的可能。

### 事件委派

在过去，web 开发者已经用在独立的单个元素上添加事件监听器来获知什么时候用户与
它们交互。例如，你会有一个 `<div>` 元素，其上注册了一个当用户点击它时触发的
函数。

尽管如此，这个途径在处理大数量交互元素上不会缩放。比如，想象一个带有 100 个
`<li>` 的 `<ul>` ，每个项目后都有一个删除按钮。既然所有的这些项目行为都是一
致的，为每个删除按钮创建共计 100 个事件监听器无疑是低效的。

<figure>
  <img src="/images/view-guide/undelegated.png">
</figure>

要解决这个问题，开发者发现了一种名为“事件委派”的技术。你可以在容器元素上注册
一个监听器并使用 `event.target` 来识别哪个元素是用户点击的，而不是为问题中的
每个项目创建一个监听器。

<figure>
  <img src="/images/view-guide/delegated.png">
</figure>

实现这有一些微妙，因为一些事件（比如 `focus` 、 `blur` 和 `change` ）不会冒
泡。幸运的是，jQuery 已经彻底解决了这个问题；用 jQuery 的 `on` 方法可以可靠
地处理所有原生浏览器事件。

其它 JavaScript 框架用两种方法中的其一来处理这个问题。第一种是，它们要你自己
实现原生解决方案，为每个项目创建独立的视图。当你创建视图，它在视图的元素上设
置一个监听器。如果你有一个含有 500 个项目的列表，你会创建 500 个视图并且每个
视图都会在它自己的元素上设置一个监听器。

第二种方法是，框架在视图层内置事件委派。当创建一个视图，你可以提供一个事件列
表来在事件发生时委派一个方法来调用。这只剩下识别接受事件的方法的点击上下文
（比如，列表中的哪个项目）。

你现在要面对两个令人不安的选择：为每个项目创建一个新视图，这样会丧失事件委派
的优势，或是为所有项目创建单个视图，这样必须存储 DOM 中底层 JavaScript 的信息。

要解决这个问题，Ember 用 jQuery 把所有事件委派到应用的根元素（通常是文档的
`body` ）。当一个事件发生，Ember 识别出最近的处理事件视图并调用它的事件处理
器。这意味着你可以创建视图来保存一个 JavaScript 上下文，但仍然从事件委派上受
益。


进一步地，因为 Ember 只为整个 Ember 应用注册一个事件，创建新视图永远都不需要
设置事件监听器，这使得重渲染高效且免于出错。当视图有一个子视图，这也意味着不
需要手动取消委派重新渲染过程中替换掉的视图。

### 渲染管道

大多数 web 应用用特殊的模板语言标记来指定它们的用户界面。对于 Ember.js，我们
已经完成用可在值修改的时候自动更新模板的 Handlebars 模板语言来编写模板。

虽然显示模板的过程对开发者是自动的，但其遮盖了把原始模板转换为最终模板、生成
用户可见的 DOM 表示的一系列必要步骤。

这是 Ember 视图的近似生命周期：

<figure>
  <img src="/images/view-guide/view-lifecycle-ember.png">
</figure>

#### 1. 模板编译

应用的模板通过网络加载或以字符串形式作为应用的载荷。当应用加载时，它发送模板
字符串到 Handlebars 来编译成函数。一经编译，模板函数会被保存，且可以被多个视
图重复使用，每次都它们都需重新编译。

这个步骤会在应用中服务器预编译模板的地方发出。在那些情况下，模板不作为原始的
人类可读的模板传输，而是编译后的代码。

因为 Ember 负责模板编译，你不需要做任何额外的工作来保证编译后的模板可以重
用。

#### 2. 字符串的连接

当应用在视图上调用 `append` 或 `appendTo` 时，一个视图渲染过程会被启动。
`append` 或 `appendChild` 调用 **安排** 视图渲染并在之后插入。这允许应用中的
延迟逻辑（譬如绑定同步）在渲染元素之前执行。

要开始渲染过程，Ember 创建一个 `RenderBuffer` 并把它呈递给视图来把视图的内容
附加到上面。在这个过程中，视图可以创建并渲染子视图。当它这么做时，父视图创建
并分配一个 `RenderBuffer` 给子视图，并把它连接到父视图的 `RenderBuffer` 上。

Ember 在渲染每个视图前刷新绑定同步队列。这样，Ember 保障不会渲染需要立即替换
的过期数据。

一旦主视图完成渲染，渲染过程会创建一个视图树（即“视图层级”），连接到缓冲区树
上。通过向下遍历缓冲区树并把它们转换为字符串，我们就有了一个可以插入到 DOM 
的字符串。

这里是一个简单的例子：

<figure>
  <img src="/images/view-guide/render-buffer.png">
</figure>

除子节点之外（字符串和其它 `RenderBuffer` ）， `RenderBuffer` 也会封装元素标
签名称、id、class、样式和其它属性。这使得渲染过程修改这些属性（例如样式）成
为可能，即使在子字符串已经渲染完毕。因为这些属性的许多都可以通过绑定（例如用
`bindAttr` ）控制，这使得渲染过程稳健且透明。

#### 3. 元素的创建和插入

在渲染过程的最后，根视图向 `RenderBuffer` 请求它的元素。 `RenderBuffer` 获得
它的完整字符串并用 jQuery 把它转换成一个元素。视图把那个元素分配到它的
`element` 属性并把把它放置到 DOM 中正确的位置（ `appendTo` 指定的位置，如果
应用使用 `append` 即是应用的根元素）。

虽然父视图直接分配它的元素，但每个子视图惰性查找它的元素。它通过查找 `id` 匹
配它的 `elementId` 属性的元素来完成这。除非显式提供，渲染过程生成一个
`elementId` 属性比你更分配它的值给视图的 `RenderBuffer` ，`RenderBuffer` 允
许视图按需查找它的元素。

#### 4. 重新渲染

在视图把自己插入到 DOM 后，Ember 和应用都会要重新渲染视图。它们可以在视图上
调用 `rerender` 方法来出发一次重渲染。

重新渲染会重复上面的步骤 2 和步骤 3，有两点例外：

* `rerender` 用新元素替换已有的元素，而不是把元素插入到显式定义的位置。
* 除了渲染新元素，它也删除旧元素并销毁它的子元素。这允许 Ember 在重新渲染视
图时自动处理撤销合适的绑定和观察者。这使得路径上的观察者可行，因为注册和撤销
注册所有的嵌套观察者都是自动的。

最常见的导致视图重新渲染的原因是当绑定到 Handlebars 表达式（ `{{foo}}` ）变
更。Ember 内部为每个表达式创建一个简单的视图，并且在路径上注册一个观察者。当
路径变更时，Ember 用新值更新那个区域的 DOM。

另一个常见的情况是一个 `{{#if}}` 或 `{{#with}}` 块。当渲染一个模板时，Ember
为这些块辅助标创建虚拟的视图。这些虚拟的视图不会出现在公共可访问的视图层级里
（当从视图获取 `parentView` 和 `childViews` 时），但它们的存在启用了一致的重
渲染。

当传递到 `{{#if}}` 或 `{{#with}}` 的路径变更，Ember 自动重新渲染虚拟视图替换
它的内容，重要的是，也会销毁所有的子视图来释放内存。

除了这些情景，应用有时也会要显式地重新渲染视图（通常是一个
`ContainerView` ，见下）。在这种情况下，应用可以直接调用 `rerender` ，且
Ember 会把一项重渲染工作加入队列，用相同的语义元素。

这个过程像是这样：

<figure>
  <img src="/images/view-guide/re-render.png">
</figure>

## 视图层级

### 父与子

当 Ember 渲染一个模板化的视图，它会生成一个视图层级。让我们假设已有一个模板
`form` 。

```handlebars
{{view App.Search placeholder="Search"}}
{{#view Ember.Button}}Go!{{/view}}
```

然后我们像这样把它插入到 DOM 中：

```javascript
var view = Ember.View.create({
  templateName: 'form'
}).append();
```

这会创建一个如下小巧的视图等级：

<figure>
  <img src="/images/view-guide/simple-view-hierarchy.png">
</figure>

你可以用 `parentView` 和 `childViews` 属性在视图层级中游走。

```javascript
var children = view.get('childViews') // [ <App.Search>, <Ember.Button> ]
children.objectAt(0).get('parentView') // 视图
```

一个常见的 `parentView` 使用方法是在子视图的实例里。

```javascript
App.Search = Ember.View.extend({
  didInsertElement: function() {
    // this.get('parentView') 指向 `view`
  }
})
```

### 生命周期钩子

为了容易地在视图的生命周期的不同点上执行行为，有若干你可以实现的钩子。

* `willInsertElement`: 这个钩子在视图渲染后插入 DOM 之前调用。它不提供对视图的 `element` 的访问。
* `didInsertElement`: 这个钩子在视图被插入到 DOM 后立即调用。它提供到视图的 `element` 的访问，且对集成到外部库非常有用。任何显式的 DOM 设置代码应限于这个钩子。
* `willDestroyElement`: 这个钩子在元素从 DOM 移除前立即调用。这提供了销毁任何与 DOM 节点关联的外部状态的机会。像 `didInsertElement` 一样，它对于集成外部库非常有用。
* `willRerender`: 这个钩子在视图被重新渲染前立即调用。如果你想要在视图被重新渲染前执行一些销毁操作，这会很有用。
* `becameVisible`: 这个钩子在视图的 `isVisible` 或它的祖先之一的 `isVisible`变为真值，且关联的元素也变为可见后调用。注意这个钩子只在所有可见性由 `isVisible` 属性控制的时候可靠。
* `becameHidden`: 这个钩子在视图的 `isVisible` 或它的祖先之一的 `isVisible`变为假值，且关联的元素也变为隐藏后调用。注意这个钩子只在所有可见性由 `isVisible` 属性控制的时候可靠。

应用可以通过在视图上定义一个与钩子同名的方法来实现钩子。或者，在视图上为钩子
注册一个监听器也是可行的。

```javascript
view.on('willRerender', function() {
  // do something with view
});
```

### 虚拟视图

正如上文所述，Handlebars 在视图层级内创建视图来表现绑定值。每次你使用
Handlebars 表达式，无论是一个简单值还是一个诸如 `{{#with}}` 或 `{{#if}}` 的
块表达式，Handlebars 会创建一个新视图。

因为 Ember 只把这些视图用于内部簿记，它们对于视图的公共 `parentView`
和 `childViews` API 是隐藏的。公共视图层级只反射用 `{{view}}` 辅助标记或通过
`ContainerView` 创建的视图（见下）。

例如，考虑下面的 Handlebars 模板：

```
<h1>Joe's Lamprey Shack</h1>
{{controller.restaurantHours}}

{{#view App.FDAContactForm}}
  如果你在乔的七鳃鳗小屋用餐后不适，请用下面的表格向 FDA 提交申诉。

  {{#if controller.allowComplaints}}
    {{view Ember.TextArea valueBinding="controller.complaint"}}
    <button {{action submitComplaint}}>提交</button>
  {{/if}}
{{/view}}
```

渲染这个模板会创建这样的层级：

<figure>
  <img src="/images/view-guide/public-view-hierarchy.png">
</figure>

幕后，Ember 跟踪为 Handlebars 表达式创建的额外的虚拟视图：

<figure>
  <img src="/images/view-guide/virtual-view-hierarchy.png">
</figure>

在 `TextArea` 中， `parentView` 会指向 `FDAContactForm` ，并且
`FDAContactForm` 的 `childViews` 会是一个只包含 `TextArea` 的数组。

你可以通过 `_parentView` 和 `_childViews` 来查看内部视图层级，这会包含虚拟视
图：

```javascript
var _childViews = view.get('_childViews');
console.log(_childViews.objectAt(0).toString());
//> <Ember._HandlebarsBoundView:ember1234>
```

**警告！** 你不应该在应用代码中依赖于这些内部 API。它们会在任何时候更改并且
没有任何公共合约。返回值也不能被观察或被绑定。它可能不是 Ember 对象。如果觉
得有使用它们的需求，请联系我们，这样我们可以为你的使用需求暴露一个更好的公共
API。

底线：这个 API 就像是 XML。如果你觉得你需要用到它，那么你很可能没有足够理解
问题。三思！

### 事件冒泡

视图的一个任务是响应原始用户事件并把它们翻译成对你应用而言有语义的事件。

例如，一个删除按钮把原始的 `click` 事件翻译成应用特定的“把这个元素从数组中删
除”。

为了响应用户事件，创建一个视图的子类来把事件实现为方法：

```javascript
App.DeleteButton = Ember.View.create({
  click: function(event) {
    var stateManager = this.getPath('controller.stateManager');
    var item = this.get('content');

    stateManager.send('deleteItem', item);
  }
});
```

当你创建一个新的 `Ember.Application` 实例，它用 jQuery 的事件委派 API 给每个
原生浏览器事件注册一个事件处理器。当用户触发一个事件，应用事件分配器会找出离
事件最近的视图并实现那个事件。

一个视图通过定义与事件同名的方法来实现事件。当事件名称由多个词组成（如
`mouseup` ）方法名会用 Camel 命名法把事件名作为方法名（ `mousUp` ）。

事件会在视图层级中冒泡，直到事件到达根视图。一个事件处理器可以用与常规
jQuery 事件处理器相同的技术来停止事件传播：

* 在视图中 `return false`
* `event.stopPropagation`

例如，假设你已经定义了如下的视图类：

```javascript
App.GrandparentView = Ember.View.extend({
  click: function() {
    console.log('Grandparent!');
  }
});

App.ParentView = Ember.View.extend({
  click: function() {
    console.log('Parent!');
    return false;
  }
});

App.ChildView = Ember.View.extend({
  click: function() {
    console.log('Child!');
  }
});
```

这是使用它们的 Handlebars 模板。

```
{{#view App.GrandparentView}}
  {{#view App.ParentView}}
    {{#view App.ChildView}}
      <h1>点击这里！</h1>
    {{/view}}
  {{/view}}
{{/view}}
```

如果你点击 `<h1>` ，你会在浏览器控制台里看见下面的输出：

```
Child!
Parent!
```

你可以看出 Ember 在接受事件的最深层级视图上调用了处理器。事件继续上浮到
`ParentView` ，但不会到达 `GrandparentView` 因为 `ParentView` 从它的事件处理
器中返回了 `false` 。

你可以使用常规事件冒泡技术来实现常见的模式。例如，你可以实现一个带有
`submit` 方法的 `FormView` 。因为浏览器在用户向文本域输入回车的时候会触发
`submit` 事件，在表单视图上定义一个 `submit` 方法会“刚好完成任务”。

```javascript
App.FormView = Ember.View.extend({
  tagName: "form",

  submit: function(event) {
    // 会在任何用户触发浏览器的
    // `submit` 方法时被调用
  }
});
```

```
{{#view App.FormView}}
  {{view Ember.TextFieldView valueBinding="controller.firstName"}}
  {{view Ember.TextFieldView valueBinding="controller.lastName"}}
  <button type="submit">确定</button>
{{/view}}
```

### 添加新事件

Ember 内置了如下原生浏览器事件的支持：

<table class="figure">
  <thead>
    <tr><th>事件名</th><th>方法名</th></tr>
  </thead>
  <tbody>
    <tr><td>touchstart</td><td>touchStart</td></tr>
    <tr><td>touchmove</td><td>touchMove</td></tr>
    <tr><td>touchend</td><td>touchEnd</td></tr>
    <tr><td>touchcancel</td><td>touchCancel</td></tr>
    <tr><td>keydown</td><td>keyDown</td></tr>
    <tr><td>keyup</td><td>keyUp</td></tr>
    <tr><td>keypress</td><td>keyPress</td></tr>
    <tr><td>mousedown</td><td>mouseDown</td></tr>
    <tr><td>mouseup</td><td>mouseUp</td></tr>
    <tr><td>contextmenu</td><td>contextMenu</td></tr>
    <tr><td>click</td><td>click</td></tr>
    <tr><td>dblclick</td><td>doubleClick</td></tr>
    <tr><td>mousemove</td><td>mouseMove</td></tr>
  </tbody>
</table>
<table class="figure">
  <thead>
    <tr><th>事件名</th><th>方法名</th></tr>
  </thead>
  <tbody>
    <tr><td>focusin</td><td>focusIn</td></tr>
    <tr><td>focusout</td><td>focusOut</td></tr>
    <tr><td>mouseenter</td><td>mouseEnter</td></tr>
    <tr><td>mouseleave</td><td>mouseLeave</td></tr>
    <tr><td>submit</td><td>submit</td></tr>
    <tr><td>change</td><td>change</td></tr>
    <tr><td>dragstart</td><td>dragStart</td></tr>
    <tr><td>drag</td><td>drag</td></tr>
    <tr><td>dragenter</td><td>dragEnter</td></tr>
    <tr><td>dragleave</td><td>dragLeave</td></tr>
    <tr><td>dragover</td><td>dragOver</td></tr>
    <tr><td>drop</td><td>drop</td></tr>
    <tr><td>dragend</td><td>dragEnd</td></tr>
  </tbody>
</table>

当你创建一个新应用时，你可以向事件分配器添加额外的事件：

```javascript
App = Ember.Application.create({
  customEvents: {
    // 添加 loadedmetadata 媒体播放器事件
    'loadedmetadata': "loadedMetadata"
  }
});
```

要使这能对自定义事件奏效，HTML5 规范必须定义事件为“bubbling”，否则 jQuery 必
须为这个事件提供一个事件委派折中方案。

## 模板化视图

如同迄今你在本指导中所见，你在应用中会用的大多数视图是依靠模板的。当使用模板
时，你不需要编写你的视图层级，因为模板会为你创建它。

渲染时，视图模板可以把视图附加到它的子视图数组中。模板的 `{{view}}` 辅助标记
内部会调用视图的 `appendChild` 方法。

调用 `appendChild` 会做两件事：

1. 把视图添加到 `childViews` 数组。
2. 立即渲染子视图并把它添加到父视图的渲染缓冲区。

<figure>
  <img src="/images/view-guide/template-appendChild-interaction.png">
</figure>

你不应该在视图离开渲染状态后调用 `appendChild` 。模板渲染出“混合内容”（包含
视图和纯文本），所以当渲染过程完成后，父视图不知道到底把新的子视图插入到哪
里。

在上例中，想象试图把一个新视图插入到父视图的 `childViews` 数组中。它应该立即
放在 `App.MyView` 的闭合标签 `</div>` 后？还是在整个视图的闭合标签 `</div>`
后？这个答案不总是正确的。

因为这种含糊性，创建视图层级的唯一方法就是用模板的 `{{view}}` 辅助标记，它总
是把视图插入到相对任何纯文本的正确位置。

虽然这个机制对大多数情景奏效，偶尔你也会想要直接程序控制一个视图的子视图。在
这种情况下，你可以用 `Ember.ContainerView` ，它显式地暴露了实现此目的的
API。

## 容器视图

容器视图不包含纯文本。它们完全由子视图（可能依靠模板）构成。

`ContainerView` 暴露两个用于修改本身内容的公共 API：

1. 一个可写的 `childViews` 数组，你可以把 `Ember.View` 实例插入到其中。
2. 一个 `currentView` 属性，设置时会把新值插入到子视图数组。如果存在早先的
   `currentView` 值，它会被从 `childViews` 数组删除。

这里是一个用 `childViews` API 创建新视图的例子，由假想的 `DescriptionView`
开始，并可以在任何时候用 `addButton` 方法添加一个新按钮：

```javascript
App.ToolbarView = Ember.ContainerView.create({
  init: function() {
    var childViews = this.get('childViews');
    var descriptionView = App.DescriptionView.create();

    childViews.pushObject(descriptionView);
    this.addButton();

    return this._super();
  },

  addButton: function() {
    var childViews = this.get('childViews');
    var button = Ember.ButtonView.create();

    childViews.pushObject(button);
  }
});
```

如你在上例中所见，我们以两个视图初始化 `ContainerView` ，并且可以在运行时添
加额外的视图。存在一个方便的捷径来设置视图，而不用覆盖 `init` 方法：

```javascript
App.ToolbarView = Ember.ContainerView.create({
  childViews: ['descriptionView', 'buttonView'],

  descriptionView: App.DescriptionView,
  buttonView: Ember.ButtonView,

  addButton: function() {
    var childViews = this.get('childViews');
    var button = Ember.ButtonView.create();

    childViews.pushObject(button);
  }
});
```

如上，当用这个速记方法时，你把 `childViews` 指定为一个字符串数组。在初始化
时，每个字符串会作为在查找视图实例或类的关键字。那个视图会被自动实例化，如果
必要，会加入到 `childViews` 数组中。

<figure>
  <img src="/images/view-guide/container-view-shorthand.png">
</figure>

## 模板作用域

标准的 Handlebars 模板有 *上下文* 的概念——出自哪个表达式的对象可以被查找到。

一些辅助标记，比如 `{{#with}}` ，在其块内修改了上下文。领一下，比如
`{{#if}}` ，会保持上下文。这些辅助标记被叫做“上下文保持辅助标记”。

当一个 Ember 应用中的 Handlebars 模板使用了一个表达式（ `{{#if foo.bar}}`
），Ember 会自动为当前上下文上的路径设置一个观察者。

如果这个路径引用的对象有变更，Ember 会自动重新以合适的上下文自动重新渲染。在
上下文保持辅助变量的情况下，Ember 会在重新渲染块时重用原始的上下文。否则，
Ember 会使用路径引用的新值作为上下文。

```
{{#if controller.isAuthenticated}}
  <h1>欢迎 {{controller.name}}</h1>
{{/if}}

{{#with controller.user}}
  <p>你有 {{notificationCount}} 条通知。</p>
{{/with}}
```

在上面的模板中，当 `isAuthenticated` 属性从 `false` 变为 `true` 时，Ember 会
重新渲染这个块，用原始的外部作用域作为它的上下文。

`{{#with}}` 辅助标记把它的块的上下文修改为当前控制器的 `user` 属性。当
`user` 属性被修改。Ember 重新渲染块，并用 `controller.user` 的新值作为它的上
下文。

### 视图作用域

除了 Handlebars 上下文，Ember 中的模板也有当前视图的概念。无论当前上下文是什
么， `view` 属性总是引用到最近的视图。

注意 `view` 属性不会引用由 `{{#if}}` 之类的块表达式创建的内部视图。这允许你
区分 Handlebars 上下文，在 Handlebars 中和在视图层级中的工作方式是一样的。

因为 `view` 指向一个 `Ember.View` 实例，你可以用 `view.propertyName` 之类的
表达式访问视图上的任何属性。你可以用 `view.parentView` 访问视图的父视图。

例如，想象你有一个带有如下属性的视图：

```javascript
App.MenuItemView = Ember.View.create({
  templateName: 'menu_item_view',
  bulletText: '*'
});
```

……和下面的模板：

```
{{#with controller}}
  {{view.bulletText}} {{name}}
{{/with}}
```

尽管 Handlebars 上下文已经变为当前的控制器，你仍然可以用 `view.bulletText`
访问视图的 `bulletText` 。

## 模板变量

迄今为止，我们已经在 Handlebars 模板中邂逅了 `controller` 属性。它是从哪来的呢？

Ember 中的 Handlebars 上下文可以继承它们的父上下文中的变量。在 Ember 在当前
上下文中查找变量之前，它首先检查它的模板变量。当一个视图创建了一个新的
Handlebars 作用域，它们自动继承它们父作用域的变量。

Ember 定义了这些 `view` 和 `controller` 变量，所以当一个表达式使用 `view` 或
`controller` 变量名，它们总是最先被找到。

如上所述，Ember 设置了 Handlebars 上下文中的 `view` 变量，无论何时模板中使用
了 `{{#view}}` 辅助标记。起初，Ember 把 `view` 变量设置为正在渲染模板的视
图。

Ember 设置了 Handlebars 上下文中的 `controller` 变量，无论已渲染的视图是否存
在 `controller` 属性。如果视图没有 `controller` 属性，它从时间上最近的拥有该
属性的视图上继承 `controller` 变量。

### 其它变量

Ember 中的 Handlebars 辅助标记也会指定变量。例如，
`{{#with controller.person as tom}}` 形式指定一个 `tom` 变量，它的后代作用域
是可访问的。即使一个子上下文有 `tom` 属性，这个 `tom` 变量会废除它。

这个形式的最大好处是，它允许你简写长路径，而不丧失对父作用域的访问权限。

在 `{{#each}}` 辅助标记中，提供 `{{#each person in people}}` 形式尤其重要。
在这个形式中，后代上下文可以访问 `person` 变量，但在模板调用 `each` 的地方
保留相同的作用域。

```
{{#with controller.preferences}}
  <h1>Title</h1>
  <ul>
  {{#each person in controller.people}}
    {{! prefix here is controller.preferences.prefix }}
    <li>{{prefix}}: {{person.fullName}}</li>
  {{/each}}
  <ul>
{{/with}}
```

注意这些变量继承了 `ContainerView` 中的那些，即使它们不是 Handlebars 上下文
层级中的一部分。

### 从视图中访问模板变量

在大多数情况下，你会需要从模板中访问这些模板变量。在一些不寻常的情景下，你会
想要在视图的 JavaScript 代码中访问范围内的变量。

你可以访问视图的 `templateVariables` 属性来达成此目的，它会返回一个包含当视
图渲染后存在于其作用于的变量的 JavaScript 对象。 `ContainerView` 也可以访问
这个属性，它指向时间上最近的模板依赖的视图的模板变量。

目前，你不能观察或绑定一个包含 `templateVariables` 的路径。
