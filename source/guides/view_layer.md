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

### Virtual Views

As described above, Handlebars creates views in the view hierarchy to
represent bound values. Every time you use a Handlebars expression,
whether it's a simple value or a block helper like `{{#with}}` or
`{{#if}}`, Handlebars creates a new view.

Because Ember uses these views for internal bookkeeping only,
they are hidden from the view's public `parentView` and `childViews`
API. The public view hierarchy reflects only views created using the
`{{view}}` helper or through `ContainerView` (see below).

For example, consider the following Handlebars template:

```
<h1>Joe's Lamprey Shack</h1>
{{controller.restaurantHours}}

{{#view App.FDAContactForm}}
  If you are experiencing discomfort from eating at Joe's Lamprey Shack,
please use the form below to submit a complaint to the FDA.

  {{#if controller.allowComplaints}}
    {{view Ember.TextArea valueBinding="controller.complaint"}}
    <button {{action submitComplaint}}>Submit</button>
  {{/if}}
{{/view}}
```

Rendering this template would create a hierarchy like this:

<figure>
  <img src="/images/view-guide/public-view-hierarchy.png">
</figure>

Behind the scenes, Ember tracks additional virtual views for the
Handlebars expressions:

<figure>
  <img src="/images/view-guide/virtual-view-hierarchy.png">
</figure>

From inside of the `TextArea`, the `parentView` would point to the
`FDAContactForm` and the `FDAContactForm`'s `childViews` would be an
array of the single `TextArea` view.

You can see the internal view hierarchy by asking for the `_parentView`
or `_childViews`, which will include virtual views:

```javascript
var _childViews = view.get('_childViews');
console.log(_childViews.objectAt(0).toString());
//> <Ember._HandlebarsBoundView:ember1234>
```

**Warning!** You may not rely on these internal APIs in application code.
They may change at any time and have no public contract. The return
value may not be observable or bindable. It may not be an Ember object.
If you feel the need to use them, please contact us so we can expose a better 
public API for your use-case.

Bottom line: This API is like XML. If you think you have a use for it,
you may not yet understand the problem enough. Reconsider!

### Event Bubbling

One responsibility of views is to respond to primitive user events
and translate them into events that have semantic meaning for your
application.

For example, a delete button translates the primitive `click` event into
the application-specific "remove this item from an array."

In order to respond to user events, create a new view subclass that
implements that event as a method:

```javascript
App.DeleteButton = Ember.View.create({
  click: function(event) {
    var stateManager = this.getPath('controller.stateManager');
    var item = this.get('content');

    stateManager.send('deleteItem', item);
  }
});
```

When you create a new `Ember.Application` instance, it registers an event
handler for each native browser event using jQuery's event delegation
API. When the user triggers an event, the application's event dispatcher
will find the view nearest to the event target that implements the
event.

A view implements an event by defining a method corresponding to the
event name. When the event name is made up of multiple words (like
`mouseup`) the method name should be the camelized form of the event
name (`mouseUp`).

Events will bubble up the view hierarchy until the event reaches the
root view. An event handler can stop propagation using the same
techniques as normal jQuery event handlers:

* `return false` from the method
* `event.stopPropagation`

For example, imagine you defined the following view classes:

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

And here's the Handlebars template that uses them:

```
{{#view App.GrandparentView}}
  {{#view App.ParentView}}
    {{#view App.ChildView}}
      <h1>Click me!</h1>
    {{/view}}
  {{/view}}
{{/view}}
```

If you clicked on the `<h1>`, you'd see the following output in your
browser's console:

```
Child!
Parent!
```

You can see that Ember invokes the handler on the child-most view that
received the event. The event continues to bubble to the `ParentView`,
but does not reach the `GrandparentView` because `ParentView` returns
false from its event handler.

You can use normal event bubbling techniques to implement familiar
patterns. For example, you could implement a `FormView` that defines a
`submit` method. Because the browser triggers the `submit` event when
the user hits enter in a text field, defining a `submit` method on the
form view will "just work".

```javascript
App.FormView = Ember.View.extend({
  tagName: "form",

  submit: function(event) {
    // will be invoked whenever the user triggers
    // the browser's `submit` method
  }
});
```

```
{{#view App.FormView}}
  {{view Ember.TextFieldView valueBinding="controller.firstName"}}
  {{view Ember.TextFieldView valueBinding="controller.lastName"}}
  <button type="submit">Done</button>
{{/view}}
```

### Adding New Events

Ember comes with built-in support for the following native browser
events:

<table class="figure">
  <thead>
    <tr><th>Event Name</th><th>Method Name</th></tr>
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
    <tr><th>Event Name</th><th>Method Name</th></tr>
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

You can add additional events to the event dispatcher when you create a
new application:

```javascript
App = Ember.Application.create({
  customEvents: {
    // add support for the loadedmetadata media
    // player event
    'loadedmetadata': "loadedMetadata"
  }
});
```

In order for this to work for a custom event, the HTML5 spec must define
the event as "bubbling", or jQuery must have provided an event
delegation shim for the event.

## Templated Views

As you've seen so far in this guide, the majority of views that you will
use in your application are backed by a template. When using templates,
you do not need to programmatically create your view hierarchy because
the template creates it for you.

While rendering, the view's template can append views to its child views
array. Internally, the template's `{{view}}` helper calls the view's
`appendChild` method.

Calling `appendChild` does two things:

1. Adds the child view to the `childViews` array.
2. Immediately renders the child view and adds it to the parent's render
   buffer.

<figure>
  <img src="/images/view-guide/template-appendChild-interaction.png">
</figure>

You may not call `appendChild` on a view after it has left the rendering
state. A template renders "mixed content" (both views and plain text) so
the parent view does not know exactly where to insert the new child view
once the rendering process has completed.

In the example above, imagine trying to insert a new view inside of
the parent view's `childViews` array. Should it go immediately
after the closing `</div>` of `App.MyView`? Or should it go after the
closing `</div>` of the entire view? There is no good answer that will
always be correct.

Because of this ambiguity, the only way to create a view hierarchy using
templates is via the `{{view}}` helper, which always inserts views
in the right place relative to any plain text.

While this works for most situations, occasionally you may want to have
direct, programmatic control of a view's children. In that case, you can
use `Ember.ContainerView`, which explicitly exposes a public API for
doing so.

## Container Views

Container views contain no plain text. They are composed entirely of
their child views (which may themselves be template-backed).

`ContainerView` exposes two public APIs for changing its contents:

1. A writable `childViews` array into which you can insert `Ember.View`
   instances.
2. A `currentView` property that, when set, inserts the new value into
   the child views array. If there was a previous value of
   `currentView`, it is removed from the `childViews` array.

Here is an example of using the `childViews` API to create a view that
starts with a hypothetical `DescriptionView` and can add a new button at
any time by calling the `addButton` method:

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

As you can see in the example above, we initialize the `ContainerView`
with two views, and can add additional views during runtime. There is a
convenient shorthand for doing this view setup without having to
override the `init` method:

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

As you can see above, when using this shorthand, you specify the
`childViews` as an array of strings. At initialization time, each of the
strings is used as a key to look up a view instance or class. That view
is automatically instantiated, if necessary, and added to the
`childViews` array.

<figure>
  <img src="/images/view-guide/container-view-shorthand.png">
</figure>

## Template Scopes

Standard Handlebars templates have the concept of a *context*--the
object from which expressions will be looked up.

Some helpers, like `{{#with}}`, change the context inside their block.
Others, like `{{#if}}`, preserve the context. These are called
"context-preserving helpers."

When a Handlebars template in an Ember app uses an expression
(`{{#if foo.bar}}`), Ember will automatically set up an
observer for that path on the current context.

If the object referenced by the path changes, Ember will automatically
re-render the block with the appropriate context. In the case of a
context-preserving helper, Ember will re-use the original context when
re-rendering the block. Otherwise, Ember will use the new value of the
path as the context.

```
{{#if controller.isAuthenticated}}
  <h1>Welcome {{controller.name}}</h1>
{{/if}}

{{#with controller.user}}
  <p>You have {{notificationCount}} notifications.</p>
{{/with}}
```

In the above template, when the `isAuthenticated` property changes from
false to true, Ember will render the block, using the original outer
scope as its context.

The `{{#with}}` helper changes the context of its block to the `user`
property on the current controller. When the `user` property changes,
Ember re-renders the block, using the new value of `controller.user` as
its context.

### View Scope

In addition to the Handlebars context, templates in Ember also have the
notion of the current view. No matter what the current context is, the
`view` property always references the closest view.

Note that the `view` property never references the internal views
created for block expressions like `{{#if}}`. This allows you to
differentiate between Handlebars contexts, which always work the way
they do in vanilla Handlebars, and the view hierarchy.

Because `view` points to an `Ember.View` instance, you can access any
properties on the view by using an expression like `view.propertyName`.
You can get access to a view's parent using `view.parentView`.

For example, imagine you had a view with the following properties:

```javascript
App.MenuItemView = Ember.View.create({
  templateName: 'menu_item_view',
  bulletText: '*'
});
```

…and the following template:

```
{{#with controller}}
  {{view.bulletText}} {{name}}
{{/with}}
```

Even though the Handlebars context has changed to the current
controller, you can still access the view's `bulletText` by referencing
`view.bulletText`.

## Template Variables

So far in this guide, we've been handwaving around the use of the
`controller` property in our Handlebars templates. Where does it come
from?

Handlebars contexts in Ember can inherit variables from their parent
contexts. Before Ember looks up a variable in the current context, it
first checks in its template variables. As a template creates new
Handlebars scope, they automatically inherit the variables from their
parent scope.

Ember defines these `view` and `controller` variables, so they are
always found first when an expression uses the `view` or `controller`
names.

As described above, Ember sets the `view` variable on the Handlebars
context whenever a template uses the `{{#view}}` helper. Initially,
Ember sets the `view` variable to the view rendering the template.

Ember sets the `controller` variable on the Handlebars context whenever
a rendered view has a `controller` property. If a view has no
`controller` property, it inherits the `controller` variable from the
most recent view with one.

### Other Variables

Handlebars helpers in Ember may also specify variables. For example, the
`{{#with controller.person as tom}}` form specifies a `tom` variable
that descendent scopes can access. Even if a child context has a `tom`
property, the `tom` variable will supersede it.

This form has one major benefit: it allows you to shorten long paths
without losing access to the parent scope.

It is especially important in the `{{#each}}` helper, which provides
the `{{#each person in people}}` form.
In this form, descendent context have access to the `person` variable,
but remain in the same scope as where the template invoked the `each`.

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

Note that these variables inherit through `ContainerView`s, even though
they are not part of the Handlebars context hierarchy.

### Accessing Template Variables from Views

In most cases, you will need to access these template variables from
inside your templates. In some unusual cases, you may want to access the
variables in-scope from your view's JavaScript code.

You can do this by accessing the view's `templateVariables` property,
which will return a JavaScript object containing the variables that were
in scope when the view was rendered. `ContainerView`s also have access
to this property, which references the template variables in the most
recent template-backed view.

At present, you may not observe or bind a path containing
`templateVariables`.
