# Ember 应用结构

在高层，你通过设计一系列符合嵌套的应用状态的嵌套的路由来组织 Ember 应用。本
指导会首先涵盖高层概念，然后用一个例子贯穿整个讲解。

## 路由

用户通过决定浏览什么来在你的应用中穿梭。例如，如果你有一个 blog，你的用户会
首先在“关于”页面和“文章列表”间选择。一般地，你想要给这个首先的选择一个默认值
（在这种情况下，可能是“文章列表”）。

一旦用户做出了他们的第一次选择，他们通常没有完成。在“文章列表”的上下文中，用
户最后会选择某篇文章和它的评论。在单篇文章页面中，他们可以在评论列表和引用通
知列表中选择。

重要的是，在所有这些情况中，用户只是在页面上显示的东西中做选择。如果你深入应
用的状态，这些选择只影响页面上的更小的区域。

在接下来的一节，我们会介绍如何控制页面上的这些区域。那么现在，让我们看看如何
构建你的模板。

当用户最开始访问到应用时，应用显示在屏幕上，并且有一个空的、路由可控制的插
座。在 Ember 中，一个 `outlet` 是模板上的一个区域，这个区域由子模板在运行时
基于用户交互来决定。

<figure>
  <img src="/images/outlet-guide/application-choice.png">
</figure>

应用模板（ `application.handlebars` ）看起来会是这样：

```
<h1>我的应用</h1>

{{outlet}}
```

默认情况下，路由起先会进入 _文章列表_ 状态，然后把插座用 `posts.handlebars`
填充。之后我们会看到这如何确切地奏效。

<figure>
  <img src="/images/outlet-guide/list-of-posts.png">
</figure>

与期待一致， _文章列表_ 模板会渲染一个文章列表。点击单篇文章的链接会用单篇文
章的模板来替换应用的插座中的内容。

模板看起来是这样：

```
{{#each post in controller}}
<h1><a {{action showPost post href=true}}>{{post.title}}</a></h1>
<div>{{post.intro}}</div>
{{/each}}
```

当点击单篇文章的连接，应用会转移到 _单篇文章_ 状态，并用 `post.handlebars`
来替换应用插座中的 `posts.handlebars` 。

<figure>
  <img src="/images/outlet-guide/individual-post.png">
</figure>

在这种情况下，单篇文章也可以有插座。插座会允许用户在评论和引用通知之间选择。

单篇文章的模板看起来是这样：

```
<h1>{{title}}</h1>

<div class="body">
  {{body}}
</div>

{{outlet}}
```

`{{outlet}}` 再次指定了路由来决定这个区域放置什么的模板。

因为 `{{outlet}}` 是所有模板的特性，当你深入路由层级，每个路由会自然地控制页
面上的更小区域。

## 它如何工作

现在你理解了基本理论，让我们看看路由是如何控制你的插座的。

### 模板、控制器以及视图

首先，对于每个高层 Handlebars 模板，你同样也会有一个同名的视图和控制器。例
如：

* `application.handlebars`: 应用主视图的模板
* `App.ApplicationController`: 上述模板的控制器。 `application.handlebars`
  的初始变量上下文是这个控制器的一个实例
* `App.ApplicationView`: 上述模板的视图对象

一般地，你会用视图对象来处理事件，并用控制器对象来向模板提供数据。

Ember 提供两种基本类型的控制器， `ObjectController` 和 `ArrayController` 。
这些控制器充当模型对象和模型对象列表的代理。

我们以控制器开始，而不是直接向模板暴露模型对象，这样你才有余地使用视图关联的
计算属性，并且最终视图关系不会污染你的模型。

你也可以用模板关联的控制器连接 `{{outlet}}` 。

### 路由

应用的路由负责让应用在状态见转移来响应用户的动作。

我们以一个简单的路由开始：

```javascript
App.Router = Ember.Router.extend({
  root: Ember.Route.extend({
    index: Ember.Route.extend({
      route: '/',
      redirectsTo: 'posts'
    }),

    posts: Ember.Route.extend({
      route: '/posts'
    }),

    post: Ember.Route.extend({
      route: '/posts/:post_id'
    })
  })
});
```

这个路由设置了三个顶层的状态：一个索引页状态。一个显示文章列表的状态和一个
显示单篇文章的状态。

在我们的案例中，我们会简单重定向索引页路由到 `posts` 状态。在其它应用中，你
也许会需要一个独立的主页。

目前为止，我们已经有了一个状态列表，并且我们的应用也会尽职尽责地进入到
`posts` 状态，但这不会做任何事。当应用进入到 `posts` 状态，我们要它连接到应
用模板中的 `{{outlet}}` 。我们用 `connectOutlets` 回调来完成这个工作。

```javascript
App.Router = Ember.Router.extend({
  root: Ember.Route.extend({
    index: Ember.Route.extend({
      route: '/',
      redirectsTo: 'posts'
    }),

    posts: Ember.Route.extend({
      route: '/posts',

      connectOutlets: function(router) {
        router.get('applicationController').connectOutlet('posts', App.Post.find());
      }
    }),

    post: Ember.Route.extend({
      route: '/posts/:post_id'
    })
  })
});
```

`connectOutlet` 调用会为我们做这些事：

* 它创建一个 `App.PostsView` 的新实例，使用 `posts.handlebars` 模板。
* 它设置 `postsController` 的 `content` 属性为一个所有可用文章（
  `App.Post.find()` ） 的列表，并让 `postsController` 作为新的
  `App.PostsView` 的控制器。
* 它把新视图连接到 `application.handlebars` 的插座上。


一般地，你应该值考虑这些对象为串联的操作。当你创建一个视图，你总是会为视图的
控制器提供内容。

## 过渡和 URL

下一步，我们要为 `posts` 状态中的应用提供一种迁移到 `post` 状态的方法。我们
通过指定一个过渡来完成这个工作。

```javascript
posts: Ember.Route.extend({
  route: '/posts',
  showPost: Ember.Route.transitionTo('post'),

  connectOutlets: function(router) {
    router.get('applicationController').connectOutlet('posts', App.Post.find());
  }
})
```

你用当前模板中的 `{{action}}` 辅助标记调用这个过渡。

```
{{#each post in controller}}
  <h1><a {{action showPost post href=true}}>{{post.title}}</a></h1>
{{/each}}
```

当用户点击一个带有 `{{action}}` 辅助标记的链接时，Ember 会把一个事件分配到指
定名称的当前状态。在这种情况下，事件是一个过渡。

因为我们使用了一个过渡，Ember 也可以为这个链接生成 URL。Ember 用上下文中的
`id` 属性来填充 `post` 状态中的 `:post_id` 动态段。

下一步，我们会需要在 `post` 状态上实现 `connectOutlets` 。这次，
`connectOutlets` 方法会接受 `{{action}}` 辅助标记上下文指定的文章对象。

```javascript
post: Ember.Route.extend({
  route: '/posts/:post_id',

  connectOutlets: function(router, post) {
    router.get('applicationController').connectOutlet('post', post);
  }
})
```

`connectOutlet` 调用执行的一系列步骤可以概括为如下：

* 它用 `post.handlebars` 模板创建了一个 `App.PostView` 的新实例。
* 它设置了用户点击的文章的 `postController` 的 `content` 属性。
* 它把新视图连接到 `application.handlebars` 中的插座。

如果用户把页面存为书签并在之后返回，你不需要任何额外的操作来让链接（ 
`/posts/1` ） 正常工作。

如果用户第一次以 `posts/1` URL 访问页面，路由会执行这几个步骤：

* 断定 URL 符合的状态（在本例中是 `post` ）。
* 从 URL 中解压动态段（在本例中是 `:post_id` ）并调用
  `App.Post.find(post_id)` 。这使用一个命名约定来奏效：
  `:post_id` 动态段对应 `App.Post` 。
* 用 `App.Post.find` 的返回值调用 `connectOutlets` 。

这意味着不管用户是否从页面中的另一部分或是通过 URL 进入到 `post` 状态，路由
都会以相同的对象调用 `connectOutlets` 方法。

## 嵌套

最后，让我们实现评论和引用通知功能。

因为 `post` 状态使用和 `root` 状态相同的模式，它看起来非常类似。

```javascript
post: Ember.Route.extend({
  route: '/posts/:post_id',

  connectOutlets: function(router, post) {
    router.get('applicationController').connectOutlet('post', post);
  },

  index: Ember.Route.extend({
    route: '/',
    redirectsTo: 'comments'
  }),

  comments: Ember.Route.extend({
    route: '/comments',
    showTrackbacks: Ember.Route.transitionTo('trackbacks'),

    connectOutlets: function(router) {
      var postController = router.get('postController');
      postController.connectOutlet('comments', postController.get('comments'));
    }
  }),

  trackbacks: Ember.Route.extend({
    route: '/trackbacks',
    showComments: Ember.Route.transitionTo('comments'),

    connectOutlets: function(router) {
      var postController = router.get('postController');
      postController.connectOutlet('trackbacks', postController.get('trackbacks'));
    }
  })
})
```

这里只发生了这些变化：

* 我们只在状态内指定了 `showTrackbacks` 和 `showComments` 过渡，而状态里过渡
  才有意义。
* 既然我们正在获取给 `post.handlebars` 使用的视图，我们调用 `postController`
  上的 `connectOutlet`
* 这种情况下，我们从当前文章中获取 `commentsController` 和
  `trackbacksController` 的内容。 `postController` 是一个底层文章模型的代
  理，所以我们可以直接用 `postController` 直接检索关联。

这是单篇文章的模板：

```
<h1>{{title}}</h1>

<div class="body">
  {{body}}
</div>

<p>
  <a {{action showComments href=true}}>评论</a> |
  <a {{action showTrackbacks href=true}}>引用通知</a>
</p>

{{outlet}}
```

最后，这个嵌套配置下，从书签链接返回页面也会正常工作。让我们看一下当用户从
`posts/1/trackbacks` 访问站点时发生了什么。

* 路由决定 URL 关联的状态（ `post.trackbacks` ），然后进入状态。
* 对经过的每个状态，路由解压任何的动态段并调用 `connectOutlets` 。这镜像了用
  户用来在应用中浏览的路径。与之前一样，路由会在文章上以 `App.Post.find(1)`
  的结果调用 `connectOutlet` 。
* 当路由进入到引用通知的状态，它会调用 `connectOutlets` 。因为 `post` 的
  `connectOutlets` 方法已经设置了 `postController` 的 `content` ，引用通知状
  态会检索关联。

再一次，由于 `connectOutlets` 回调与动态 URL 段协同工作，由 `{{action}}` 辅
助标记生成的 URL 会之后会保证奏效。

## 异步

最后一点：你会问你自己，当应用在 `App.Post.find(1)` 调用时还没有加载“文章1”
这个系统如何正常工作。

这会奏效的原因是 `ember-data` 总是立即返回一个对象，即使它需要开启一个查询。
那个对象以一个空的 `data` 散列值开始。当服务器返回数据， `ember-data` 更新对
象的 `data` ，这也出发所有定义的属性（用 `DS.attr` 定义的属性）上的绑定。

当你要这个对象查询它的 `trackbacks` ，它也会返回一个空的 `ManyArray` 。当服
务器一同返回文章和与之关联的内容时， `ember-data` 会自动更新 `trackbacks` 数
组。

在你的 `trackbacks.handlebars` 模板中，你会做好这些：

```
<ul>
{{#each trackback in controller}}
  <li><a {{bindAttr href="trackback.url"}}>{{trackback.title}}</a></li>
{{/each}}
</ul>
```

当 `ember-data` 更新 `trackbacks` 数组，变更会通过 `trackbacksController` 传
播至 DOM。

你也会想要避免展示尚未加载的局部数据。在这种情况下，你可以这么做：

```
<ul>
{{#if controller.isLoaded}}
  {{#each trackback in controller}}
    <li><a {{bindAttr href="trackback.url"}}>{{trackback.title}}</a></li>
  {{/each}}
{{else}}
  <li><img src="/spinner.gif">加载引用通知……</li>
{{/if}}
</ul>
```

当 `ember-data` 用服务器提供的数据把引用通知填入到 `ManyArray` 里，它也会设
置 `isLoaded` 属性。因为所有的包含 `#if` 的模板结构会在底层属性变化时自动更
新 DOM，这会“恰好奏效”。
