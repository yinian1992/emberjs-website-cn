## 用 Handlebars 描述你的 UI

### Handlebars

Ember 捆绑了 [Handlebars](http://www.handlebarsjs.com)，一个语义化的
模板语言。这些模板看起来像是嵌入了表达式的普通 HTML。

你应该把 Handlebars 模板存储在应用的 HTML 文件里。在运行时，Ember 会
编译这些模板，之后它们会在视图中可用。

要立即向文档中插入一个模板，把它放置在 `<body>` 标签里的一个 `<script>`
标签里：

```handlebars
<html>
  <body>
    <script type="text/x-handlebars">
      Hello, <b>{{MyApp.name}}</b>
    </script>
  </body>
</html>
```

要让模板在后面可以使用，给定 `<script>` 标签的 `data-template-name`
属性：

```handlebars
<html>
  <head>
    <script type="text/x-handlebars" data-template-name="say-hello">
      Hello, <b>{{MyApp.name}}</b>
    </script>
  </head>
</html>
```

### Ember.View

你可以使用 `Ember.view` 来渲染 Handlebars 模板并把它插入到 DOM 中。

设置它的 `templateName` 属性来告知视图使用哪个模板。例如，如果我已
经有一个这样的 `<script>` 标签：

```handlebars
<html>
  <head>
    <script type="text/x-handlebars" data-template-name="say-hello">
      Hello, <b>{{name}}</b>
    </script>
  </head>
</html>
```

我会把 `templateName` 属性设置为 `"say-hello"` 。

```javascript
var view = Ember.View.create({
  templateName: 'say-hello',
  name: "Bob"
});
```

注意：在指导剩下的部分中， `templateName` 属性会在大多数例子中省略。
你可以认为当我们展示一个包含 Ember.view 和 Handlebars 模板的代码示例
时，视图已经被配置好显示 `templateName` 属性给定的模板。

你可以调用 `appendTo` 来把视图附加到文档后：

```javascript
  view.appendTo('#container');
```

你可以用 `append` 这个快捷方式来把视图附加到文档的 body 后：

```javascript
  view.append();
```

调用 `remove` 从文档删除一个视图：

```javascript
  view.remove();
```

### Handlebars 基础

如你所见，你可以打印 Handlebars 表达式或是一组括号中的属性值，像这样：

```handlebars
My new car is {{color}}.
```

这会查找并打印出视图中的 `color` 属性。例如，如果你的视图是这样：

```javascript
App.CarView = Ember.View.extend({
  color: 'blue'
});
```

你的视图在浏览器中会显示为：

```html
My new car is blue.
```

同样也可以使用全局路径：

```handlebars
My new car is {{App.carController.color}}.
```

（Ember 通过检查首字母大写来确定路径是全局的还是相对的，这就是为什么
`Ember.Application` 实例开头字母应大写。）

本指导中提到的所有特性都是 __绑定可知的__ 。这意味着如果模板中使用的
值发生变化，你的 HTML 也会自动更新。妙不可言。

当底层属性变更时，为了获知更新哪部分 HTML，Handlebars 会插入唯一 ID 的
标识元素。如果你查看一个运行中的应用，你会可能注意到这些额外的元素：

```html
My new car is
<script id="metamorph-0-start" type="text/x-placeholder"></script>
blue
<script id="metamorph-0-end" type="text/x-placeholder"></script>.
```

因为所有的 Handlebars 表达式都被包裹在这些标识中，确保每个 HTML 标签都
在同一个块中。例如，你不应该这样：

```handlebars
<!-- 求别这样！ -->
<div {{#if isUrgent}}class="urgent"{{/if}}>
```

如果你想避免属性输出被包裹在这些标识中，使用 `unbound` 辅助标记：

```handlebars
My new car is {{unbound color}}.
```

你的数据就远离标识，但是注意这些输出不会再自动更新！

```html
My new car is blue.
```

### {{#if}}、 {{else}} 和 {{#unless}}

有时，当一个属性存在，你只是想要显示模板中的部分。例如，让我们假设我们
已经有一个带有 `person` 属性的视图，这个属性包含了一个有 `firstName` 和
`lastName` 字段的对象：

```javascript
App.SayHelloView = Ember.View.extend({
  person: Ember.Object.create({
    firstName: "Joy",
    lastName: "Clojure"
  })
});
```

要仅在 `person` 对象存在时仅显示模板的一部分，我们可以用 `{{#if}}` 辅助
标记来条件性地渲染一个块：

```handlebars
{{#if person}}
  Welcome back, <b>{{person.firstName}} {{person.lastName}}</b>!
{{/if}}
```

如果传递的参数求值为 `false` 、 `undefined` 、 `null` 或是 `[]` （也就是
任何“假”值），Handlebars 不会渲染块。

如果表达式值为假，我们也可以用 `{{else}}` 显示一个可选的模板：

```handlebars
{{#if person}}
  Welcome back, <b>{{person.firstName}} {{person.lastName}}</b>!
{{else}}
  Please log in.
{{/if}}
```

要在一个值为假，只渲染一个块，用 `{{#unless}}` ：

```handlebars
{{#unless hasPaid}}
  You owe: ${{total}}
{{/unless}}
```

`{{#if}}` 和 `{{#unless}}` 是块表达式的例子。这些会允许你在模板的部分
中调用一个辅助标记。块表达式如同常规表达式，只是在辅助标识名中包含一个
井号（#），并需要一个闭合的表达式。

### {{#with}}

有时你会需要以一个与 Ember.View 中不同的上下文来调用模板中的某节。例如，
我们可以用 `{{#with}}` 辅助标记来清理上面的文档：

```handlebars
{{#with person}}
  Welcome back, <b>{{firstName}} {{lastName}}</b>!
{{/with}}
```

`{{#with}}` 改变你传给它的块的 _上下文_ 。上下文是查找属性的那个对象。默认
情况下，上下文是模板所属的 Ember.View。

### 用 {{bindAttr}} 绑定元素属性

除了文本，你也可能会想要模板支配 HTML 元素的属性。例如，想象包含一个 URL 的
视图：

```javascript
App.LogoView = Ember.View.extend({
  logoUrl: 'http://www.mycorp.com/images/logo.png'
});
```

在 Handlebars 中显示 URL 为图像的最佳方式是这样的：

```handlebars
<div id="logo">
  <img {{bindAttr src="logoUrl"}} alt="Logo">
</div>
```

这会生成下面的 HTML：

```html
<div id="logo">
  <img src="http://www.mycorp.com/images/logo.png" alt="Logo">
</div>
```

如果你对一个布尔值使用 `{{bindAttr}} ，它会添加或删除指定的属性。例如，下
面的 Ember 视图：

```javascript
App.InputView = Ember.View.extend({
  isDisabled: true
});
```

还有这个模板：

```handlebars
<input type="checkbox" {{bindAttr disabled="isDisabled"}}>
```

Handlebars 会生成下面的 HTML 元素：

```html
<input type="checkbox" disabled>
```

### 用 {{bindAttr}} 绑定 class 名

`class` 属性可以像其它属性一样绑定，但是它也有一些额外的特殊行为。默认的
行为会如你所愿：

```javascript
App.AlertView = Ember.View.extend({
  priority: "p4",
  isUrgent: true
});
```

```handlebars
<div {{bindAttr class="priority"}}>
  Warning!
</div>
```

这个模板会发布成下面的 HTML：

```html
<div class="p4">
  Warning!
</div>
```

如果你绑定的值是布尔值，无论如何，会把属性做“-”变换作为 class 名：

```handlebars
<div {{bindAttr class="isUrgent"}}>
  Warning!
</div>
```

这会发出下面的 HTML：

```html
<div class="is-urgent">
  Warning!
</div>
```

不像其它属性，你可以绑定多个 class：

```handlebars
<div {{bindAttr class="isUrgent priority"}}>
  Warning!
</div>
```

你也可以指定一个可选的 class 名，而不只是进行“-”变换。

```handlebars
<div {{bindAttr class="isUrgent:urgent"}}>
  Warning!
</div>
```

在这种情况下，如果 `isUrgent` 为真，则会把 `urgent` 作为 class。反之，
`urgent` 不会作为 class。

你也可以指定在属性为 `false` 时使用的 class 名。

```handlebars
<div {{bindAttr class="isEnabled:enabled:disabled"}}>
  Warning!
</div>
```

在这种情况，如果 `isEnabled` 属性为真，会添加 `enabled` 作为 class 。
反之会使用 `disabled` 。

这个语法允许快捷地只在属性为 `false` 是添加一个 class，那么这样：

```handlebars
<div {{bindAttr class="isEnabled::disabled"}}>
  Warning!
</div>
```

当 `isEnabled` 为 `false` 时会添加 `disabled` 作为 class，而在
`isEnabled` 为 `true` 时不添加任何 class。

### 用 {{action}} 处理事件

用 `{{action}}` 辅助标记来在把你的视图类中的一个处理器附加到一个在某元素上触
发的事件上。

要把某元素的 `click` 事件附加到当前视图的 `edit()` 处理器：

```handlebars
<a href="#" {{action "edit" on="click"}}>Edit</a>
```

因为默认事件是 `click` ，这也可以简写作：

```handlebars
<a href="#" {{action "edit"}}>Edit</a>
```

虽然包含 `{{action}}` 辅助标记的视图会作为默认目标，但也可以指定其它的视图作
为目标：

```handlebars
<a href="#" {{action "edit" target="parentView"}}>Edit</a>
```

动作处理器也可以接受一个被扩展为包含 `view` 和 `context` 属性的 jQuery 事件
对象。这些属性在把其它视图作为动作目标时会用到。例如：

```javascript
App.ListingView = Ember.View.extend({
  templateName: 'listing',

  edit: function(event) {
    event.view.set('isEditing', true);
  }
});
```

上述的任意模板都会生成这样的 HTML 元素：

```html
<a href="#" data-ember-action="3">Edit</a>
```

Ember 会基于内部分配的 `data-ember-action` id 来委托你给定的事件到你的目标视
图处理器。

### 构建层次视图

迄今为止，我们已经讨论了编写单一视图的模板。然而，随着你的应用发展，你会经常
需要构建分层的视图来封装页面上的不同部分。每个视图在处理事件和维护需要显示的
属性上都是可靠的。

### {{view}}

用 `{{view}}` 辅助标记来为父视图添加一个子视图，它需要一个到视图类的路径。

```javascript
// 定义父视图
App.UserView = Ember.View.extend({
  templateName: 'user',

  firstName: "Albert",
  lastName: "Hofmann"
});

// 定义子视图
App.InfoView = Ember.View.extend({
  templateName: 'info',

  posts: 25,
  hobbies: "Riding bicycles"
});
```

```handlebars
User: {{firstName}} {{lastName}}
{{view App.InfoView}}
```

```handlebars
<b>Posts:</b> {{posts}}
<br>
<b>Hobbies:</b> {{hobbies}}
```

如果你要创建一个 `App.UserView` 的实例并渲染它，我们会得到这样的 DOM 表示：

```html
User: Albert Hofmann
<div>
  <b>Posts:</b> 25
  <br>
  <b>Hobbies:</b> Riding bicycles
</div>
```

#### 相对路径

除了指定一个绝对路径，你也可以指定使用哪个相对于父视图的视图类。例如，我们可
以这样嵌套上面的视图层级：

```javascript
App.UserView = Ember.View.extend({
  templateName: 'user',

  firstName: "Albert",
  lastName: "Hofmann",

  infoView: Ember.View.extend({
    templateName: 'info',

    posts: 25,
    hobbies: "Riding bicycles"
  })
});
```

```handlebars
User: {{firstName}} {{lastName}}
{{view infoView}}
```

当这样嵌套一个视图类，确保使用小写字母，因为 Ember 会把大写字母的属性视为全
局属性、

### 设置子视图模板

如果你想在主模板中内联地指定子视图使用的模板，你可以用块形式的 `{{view}}` 辅
助标记。你可以这样重写上面的例子：

```javascript
App.UserView = Ember.View.extend({
  templateName: 'user',

  firstName: "Albert",
  lastName: "Hofmann"
});

App.InfoView = Ember.View.extend({
  posts: 25,
  hobbies: "Riding bicycles"
});
```

```handlebars
User: {{firstName}} {{lastName}}
{{#view App.InfoView}}
  <b>Posts:</b> {{view.posts}}
  <br>
  <b>Hobbies:</b> {{view.hobbies}}
{{/view}}
```

当你这么做时，会对有助于把它认为是分配视图到页面的部分。这允许你封装那部分页
面的事件处理。

### 设置绑定

目前为止，在我们的例子中一直在视图中直接设置静态值。而为了 MVC 架构的最佳实
现，我们实际上应该绑定视图中的属性到控制器层。

让我们设置一个控制器来表示我们的用户数据：

```javascript
App.userController = Ember.Object.create({
  content: Ember.Object.create({
    firstName: "Albert",
    lastName: "Hofmann",
    posts: 25,
    hobbies: "Riding bicycles"
  })
});
```

现在让我们更新 `App.UserView` 来绑定到 `App.userController`：

```javascript
App.UserView = Ember.View.extend({
  templateName: 'user',

  firstNameBinding: 'App.userController.content.firstName',
  lastNameBinding: 'App.userController.content.lastName'
});
```

当我们只有少数绑定需要配置，像 `App.UserView` ，它有时候对在模板中声明那些绑
定很有用。你可以传递额外的参数给 `{{#view}}` 辅助标记来实现。如果你要做的只
是配置绑定，那么这通常让你忽视必须创建一个子类。

```handlebars
User: {{firstName}} {{lastName}}
{{#view App.UserView postsBinding="App.userController.content.posts"
        hobbiesBinding="App.userController.content.hobbies"}}
  <b>Posts:</b> {{view.posts}}
  <br>
  <b>Hobbies:</b> {{view.hobbies}}
{{/view}}
```

注意：你实际上可以传递 __任何__ 属性作为 `{{view}}` 的参数，而不只是绑定。尽
管如此，如果你在做设置绑定之外的事，创建一个新的子类通常是个好主意。

### 修改视图的 HTML

当你附加一个视图，它会创建一个新的 HTML 元素来包裹它的内容。如果你的视图有任
何子视图，它们也会显示为父视图的 HTML 元素的子节点。

默认， `Ember.View` 的新势力创建一个 `<div>` 元素。你可以传递一个 `tagName`
参数来覆盖这：

```handlebars
{{view App.InfoView tagName="span"}}
```

你同样也可以传递 `id` 参数来给视图的 HTML 元素分配一个 id 属性：

```handlebars
{{view App.InfoView id="info-view"}}
```

这使得用 CSS ID 进行样式化非常容易：

```css
/** 给视图一个红色背景 **/
  #info-view {
    background-color: red;
  }
```

你同样也可以分配 class：

```handlebars
{{view App.InfoView class="info urgent"}}
```

你可以用 `classBinding` 而不是 `class` 来把 class 名绑定到视图的属性上。其行
为与 `bindAttr` 中描述的一样：

```javascript
App.AlertView = Ember.View.extend({
  priority: "p4",
  isUrgent: true
});
```

```handlebars
{{view App.AlertView classBinding="isUrgent priority"}}
```

这会生成一个看起来是这样的视图容器：
This yields a view wrapper that will look something like this:

```html
<div id="sc420" class="sc-view is-urgent p4"></div>
```

### 显式一个元素列表

如果你需要迭代一个对象列表，请使用 Handlebars 的 `{{#each}}` 辅助标记：

```javascript
App.PeopleView = Ember.View.extend({
  people: [ { name: 'Yehuda' },
            { name: 'Tom' } ]
});
```

```handlebars
<ul>
  {{#each people}}
    <li>Hello, {{name}}!</li>
  {{/each}}
</ul>
```

这会打印这样的列表：

```html
<ul>
  <li>Hello, Yehuda!</li>
  <li>Hello, Tom!</li>
</ul>
```

如果你想要为列表中的每个元素创建一个视图，只需要像下面这样设置：

```handlebars
{{#each App.peopleController}}
  {{#view App.PersonView}}
    {{firstName}} {{lastName}}
  {{/view}}
{{/each}}
```

### 编写自定义辅助标记

有时，你会在应用中多次用到相同的 HTML。在这些情况下，你可以注册一个可以在任
何 Handlebars 模板中调用的自定义辅助标记。

例如，想象你经常把值包裹在一个自定义 class 的 `<span>` 中。你可以像这样从
JavaScript 中注册一个辅助标记：

```javascript
Handlebars.registerHelper('highlight', function(property, options) {
  var value = Ember.Handlebars.getPath(this, property, options);
  return new Handlebars.SafeString('<span class="highlight">'+value+'</span>');
});
```

如果你从一个辅助标记返回 HTML，且不想返回的 HTML 被转义，确保返回一个新的
`SafeString` 实例。

现在，在你的 Handlebars 模板中的任何地方，你都可以调用这个辅助标记：

```handlebars
{{highlight name}}
```

并且会输出下面的东西：

```html
<span class="highlight">Peter</span>
```

注意：传递给辅助标记的参数应该是变量名，而不是实际值。这允许你在值上设置观察
者。要获取参数的实际值，像上面展示的那样使用 `Ember.getPath` 。

### 包含的视图

Ember 预包装了一系列用于构建诸如文本框、单选框和选择列表等基本控件的视图。

它们是：

####Ember.Checkbox
	
```handlebars
    <label>
      {{view Ember.Checkbox checkedBinding="content.isDone"}}
      {{content.title}}
    </label>
```
	
####Ember.TextField
	
```javascript
	App.MyText = Ember.TextField.extend({
	    formBlurredBinding: 'App.adminController.formBlurred',
	    change: function(evt) {
	      this.set('formBlurred', true);
	    }
	  });
```
	
####Ember.Select
	
```handlebars
	{{view Ember.Select viewName="select"
                          contentBinding="App.peopleController"
                          optionLabelPath="content.fullName"
                          optionValuePath="content.id"
                          prompt="Pick a person:"
                          selectionBinding="App.selectedPersonController.person"}}
```
	
####Ember.TextArea
	
```javascript
	var textArea = Ember.TextArea.create({
      		valueBinding: 'TestObject.value'
    		});
```
	

如果你想要把这些控件中的某个加入到你的视图，我们鼓励你继承这些控件。

事件不会从子视图冒泡到父视图，所以继承这些视图是捕捉那些事件的唯一方法。

例如：

```javascript
App.MyText = Ember.TextField.extend({
    formBlurredBinding: 'App.adminController.formBlurred',
    change: function(evt) {
      this.set('formBlurred', true);
    }
  });
```

你可以之后用这个视图作为子视图来捕获事件。在下面的例子中，Name 文本框的变更
会让表单失去焦点，并显示保存按钮。

```handlebars
<script id="formDetail" data-template-name='formDetail' type="text/x-handlebars">
    <form>
        <fieldset>
           <legend>Info:</legend>                 
           
                   {{view App.MyText name="Name" id="Name"  valueBinding="myObj.Name"}} 
	               <label for="Name">Name</label><br/>
                   
                   {{#if formBlurred}}
                    <a href="#" {{action "syncData" on="click"}}>Save</a>
                    {{/if}}
               
        </fieldset>
    </form>
</script>
```
