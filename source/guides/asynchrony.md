# 在 Ember 中处理异步

许多 Ember 的概念，比如绑定和计算属性，其设计是为了完成异步行为的处理。

## 没有 Ember 的情况

我们首先来看一看用 jQuery 或基于事件的 MVC 框架如何管理异步行为。

让我们使用一个 web 应用中最常见的异步行为——发起一个 Ajax 请求——来作为例子。
浏览器发起 Ajax 请求的 API 提供了一个异步的 API。jQuery 包装器也可以实现：

```javascript
jQuery.getJSON('/posts/1', function(post) {
  $("#post").html("<h1>" + post.title + "</h1>" +
    "<div>" + post.body + "</div>");
});
```

在纯 jQuery 应用中，你会用这个回调来随意进行要更改到 DOM 的更改。

当使用一个基于事件的 MVC 框架，会把逻辑从回调中拿出而放进模型和视图对象里。
这改进了很多东西，但仍未摆脱显式处理异步回调的需求：

```javascript
Post = Model.extend({
  author: function() {
    return [this.salutation, this.name].join(' ')
  },

  toJSON: function() {
    var json = Model.prototype.toJSON.call(this);
    json.author = this.author();
    return json;
  }
});

PostView = View.extend({
  init: function(model) {
    model.bind('change', this.render, this);
  },

  template: _.template("<h1><%= title %></h1><h2><%= author %></h2><div><%= body %></div>"),

  render: function() {
    jQuery(this.element).html(this.template(this.model.toJSON());
    return this;
  }
});

var post = Post.create();
var postView = PostView.create({ model: post });
jQuery('#posts').append(postView.render().el);

jQuery.getJSON('/posts/1', function(json) {
  // set all of the JSON properties on the model
  post.set(json);
});
```

这个例子没有用任何特殊的 JavaScript 库，但它的实现途径是事件驱动的 MVC 框架
中的典型。它实现了异步事件的组织，但异步行为仍然在核心程序模型中。

## Ember 的方法

总体而言，Ember 的目的是消除显式形式的异步行为。如我们之后会见到的，这给予了
Ember 合并多个具有相同结果事件的能力。

它也提供了高层抽象，消灭手动注册/反注册事件监听器来执行大多数任务的需求。

你一般会为这个例子使用用 `ember-data` ，当让我们看看如何用 jQuery 来为 Ember
中的 Ajax 模型化上面的例子。

```javascript
App.Post = Ember.Object.extend({
  
});

App.PostController = Ember.ObjectController.extend({
  author: function() {
    return [this.get('salutation'), this.get('name')].join(' ');
  }.property('salutation', 'name')
});

App.PostView = Ember.View.extend({
  // the controller is the initial context for the template
  controller: null,
  template: Ember.Handlebars.compile("<h1>{{title}}</h1><h2>{{author}}</h2><div>{{body}}</div>")
});

var post = App.Post.create();
var postController = App.PostController.create({ content: post });

App.PostView.create({ controller: postController }).appendTo('body');

jQuery.getJSON("/posts/1", function(json) {
  post.setProperties(json);
});
```

与上面的例子相反，Ember 的实现方法消灭了当 `post` 的属性变更时显式注册观察者
的需求。

`{{title}}` 、 `{{author}}` 和 `{{body}}` 模板元素被限定到 `PostController`
上的那些元素中。当 `PostController` 的内容更改，它自动传播那些变更到 DOM。

为 `author` 使用一个计算属性消灭了在底层属性变更时显式地调用回调中计算的需
求。

除此之外，Ember 的绑定系统自动跟踪从 `getJSON` 回调中设置的 `salutation` 和
`name` 到 `PostController` 中的计算属性，并且始终把变化传播到 DOM 中。

## 益处

因为 Ember 经常负责传播变更，它可以保证在响应每个用户事件上一个变更只传播一
次。

让我们再看一看 `author` 计算属性。

```javascript
App.PostController = Ember.ObjectController.extend({
  author: function() {
    return [this.get('salutation'), this.get('name')].join(' ');
  }.property('salutation', 'name')
});
```

因为我们已经指定了它依赖于 `salutation` 和 `name` ，这两个依赖的任何一个的变
更都会使属性无效，这会触发对 DOM 中 `{{author}}` 的更新。

想象在响应一个用户事件时，我要做这些事：

```javascript
post.set('salutation', "Mrs.");
post.set('name', "Katz");
```

你会臆断这些变更会导致计算属性被无效两次，导致更新 DOM 两次。而事实上，这在
使用事件驱动的框架上确实会发生。

在 Ember 中，计算属性会只重新计算一次，并且 DOM 也只会更新一次。

这是如何实现的呢？

当你对 Ember 中的一个属性做出更改，它不会立即传播变更。除此之外，它立即无效
任何有依赖的属性，但把实际的修改放入队列来让它在之后发生。

对 `salutation` 和 `name` 属性都修改会无效 `author` 属性两次，但队列会智能地
合并那些变更。

一旦所有当前用户事件的事件处理器完成，Ember 刷新队列，把变更向下传播。在这种
情况下，这意味着被无效的 `author` 属性会无效 DOM 中的 `{{author}}` ，这会让
单次请求只重新计算信息并更新自己一次。

**这个机制是 Ember 的根基。** 在 Ember 中，应该总是假定一个你所做变更的副作
用会在之后发生。通过做这个假设，你允许 Ember 来合并单次调用中相同的副作用。

总而言之，事件驱动系统的目标是从监听器产生的负效用中解耦数据操作，所以你不应
该假定同步的副作用，即使在一个更关注事件的系统中。事实上，在 Ember 中副作用
不会立刻传播，消除了欺骗并偶然地把应该分开的代码耦合在一起的诱因。

## 副作用回调

既然你不能依赖同步的副作用，你会好奇如何确保特定的行为在恰好的时间发生。

例如，想象你有一个包含一个按钮的视图，并且你想用 jQuery UI 来样式化这个按
钮。因为视图的 `append` 方法，如同 Ember 中的其它东西，推迟了它的副作用，怎
样在正确的时间执行 jQuery UI 代码？

答案是生命周期回调。

```javascript
App.Button = Ember.View.extend({
  tagName: 'button',
  template: Ember.Handlebars.compile("{{view.title}}"),

  didInsertElement: function() {
    this.$().button();
  }
});

var button = App.Button.create({
  title: "Hi jQuery UI!"
}).appendTo('#something');
```

这种情况下，一旦按钮真正出现在 DOM 中，Ember 会触发 `didInsertElement` 回
调，然后你可以做任何你想要做的工作。

生命周期回调方法有很多好处，即使我们并不需要担忧延迟的插入。

首先，依赖同步的插入意味着 `appendTo` 的调用者要来触发任何需要在附加元素后立
即运行的行为。当你的应用变大后，你会发现在许多地方创建相同的视图，并且需要担
心它对每个地方的联系。

生命周期回调消灭了实例化视图和它的提交插入行为两部分代码的耦合。一般地，我们
发现不依赖于同步副作用导致了整体上的优良设计。

第二，因为关于视图生命周期的一切都在视图本身内部，按需重渲染 DOM 的部分对
Ember 是非常容易的。

例如，如果这个按钮在一个 `{{#if}}` 块中，并且 Ember 需要从主分支切换到
`else` 节，Ember 可以轻易实例化视图并调用生命周期回调。

因为 Ember 强迫你定义一个完整定义的视图，它可以控制在合适的场合创建并插入视
图。

这也意味着所有的与 DOM 相关的代码只在应用中封装好的几个部分，所以 Ember 在
这些在回调之外的渲染过程中的部分有更多的自由。

## 观察者

在一些罕见的情况，你会想要在属性变更已经传播之后执行特定的行为。正如前面一节
中所述，Ember 提供了一个机制来挂钩到属性变更通知。

让我们返回“称呼”的例子。

```javascript
App.PostController = Ember.ObjectController.extend({
  author: function() {
    return [this.get('salutation'), this.get('name')].join(' ');
  }.property('salutation', 'name')
});
```

如果我们想在作者变更时被通知，我们可以注册一个观察者。让我们表示为视图函数
想要被通知：

```javascript
App.PostView = Ember.View.extend({
  controller: null,
  template: Ember.Handlebars.compile("<h1>{{title}}</h1><h2>{{author}}</h2><div>{{body}}</div>"),

  authorDidChange: function() {
    alert("New author name: " + this.getPath('controller.author'));
  }.observes('controller.author')
});
```

Ember 在它成功传播变更后触发观察者。在本例中，这意味着 Ember 会只对每个用户
事件调用一次 `authorDidChange` ，即使 `salutation` 和 `name` 都有变更。

这为你提供了在属性变更后执行代码的便利，而不强制所有的属性变更同步。这基本上
意味着如果你需要对一个计算属性的变更做一些手动工作，你会得到相同的合并便利，
如同在 Ember 的绑定系统中一样。

最后，你也可以手动注册观察者，在对象定义之外：

```javascript
App.PostView = Ember.View.extend({
  controller: null,
  template: Ember.Handlebars.compile("<h1>{{title}}</h1><h2>{{author}}</h2><div>{{body}}</div>"),

  didInsertElement: function() {
    this.addObserver('controller.name', function() {
      alert("New author name: " + this.getPath('controller.author'));
    });
  }
});
```

尽管如此，当你使用对象定义语法，Ember 会自动在对象销毁时销毁观察期。例如，一
个 `{{#if}}` 语句从真值变为假值，Ember 销毁块内定义的所有试图。作为这个过程
的一部分，Ember 也会断开所有绑定和内联观察者。

如果你手动定义了一个观察者，你需要确保移除了它。一般地，你会想要在与创建时相
反的回调中移除观察者。在本例中，你会想要移除 `willDestroyElement` 的回调。

```javascript
App.PostView = Ember.View.extend({
  controller: null,
  template: Ember.Handlebars.compile("<h1>{{title}}</h1><h2>{{author}}</h2><div>{{body}}</div>"),

  didInsertElement: function() {
    this.addObserver('controller.name', function() {
      alert("New author name: " + this.getPath('controller.author'));
    });
  },

  willDestroyElement: function() {
    this.removeObserver('controller.name');
  }
});
```

如果你在 `init` 方法中添加了观察者，你会想要在 `willDestroy` 回调中销毁它。

总而言之，你会极少用这种方法注册一个手动的观察者。为了保障内存管理，我们强烈
建议尽可能把观察者定义为对象定义的一部分。
