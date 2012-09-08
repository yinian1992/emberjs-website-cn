## 深入视图

现在你已经熟悉了 Handlebars 的用法，让我们深入如何处理事件和按需定制视图。

### 事件处理

只需简单实现一个与你要响应事件同名的视图方法，而不是在元素上注册一个事件监听
器。

例如，假设我们有这样一个模板：

```handlebars
{{#view App.ClickableView}}
This is a clickable area!
{{/view}}
```

让我们实现 `App.ClickableView` ，这样当点击它时，会显示一个警告：

```javascript
App.ClickableView = Ember.View.extend({
  click: function(evt) {
    alert("ClickableView was clicked!");
  }
});
```

事件会从目标视图连续冒泡到每个父视图，直到根视图。这些值会是只读的。如果你想
要在 JavaScript （而不是用 Handlebars 中的 `{{view}}` 辅助标记创建它们）中手
动管理它们，详见下面的 `Ember.ContainerView` 文档。

### 用 Ember.ContainerView 手动管理视图

通常，视图用 `{{view}}` 辅助标记来创建子视图。又是，手动管理视图的子视图是很
有效的。如果你创建一个 `Ember.ContainerView` 的实例， `childViews` 数组会可
编辑。你添加的视图将渲染到页面，并且你删除的视图也会从 DOM 中移除。

```javascript
var container = Ember.ContainerView.create();
container.append();

var coolView = App.CoolView.create(),
    childViews = container.get('childViews');

childViews.pushObject(coolView);
```

简写方案是，把子视图作为属性或是子视图作为列表的键。当容器视图创建后，这些视
图也会被实例化并且加入到子视图数组中：

```javascript
var container = Ember.ContainerView.create({
  childViews: ['firstView', 'secondView'],
  
  firstView: App.FirstView,
  secondView: App.SecondView
});
```

### 渲染管道

在你的视图转换成 DOM 元素钱，它们首先表示为字符串。当视图渲染时，它们把每个
子视图转换为字符串并连接它们。

如果你想用 Handlebars 之外的东西，你可以覆盖视图的 `render` 方法来生成自定义
的 HTML 字符串。

```javascript
App.CoolView = Ember.View.create({
  render: function(buffer) {
    buffer.push("<b>This view is so cool!</b>");
  }
});
```

这使得支持 Handlebars 之外的模板引擎非常容易；只是注意，如果你覆盖了渲染方法，
值不会自动更新。任何更新都是你的任务了。

### 自定义 HTML 元素

视图会表现为页面上的一个 DOM 元素。你可以改变 `tagName` 属性来决定要创建哪种
元素。

```javascript
App.MyView = Ember.View.extend({
  tagName: 'span'
});
```

你也可以通过设置 `classNames` 属性指定应用到视图的 class 名，它是一个字符串
的数组：

```javascript
App.MyView = Ember.View.extend({
  classNames: ['my-view']
});
```

如果你想让 class 名由视图的状态来决定，你可以使用 class 名绑定。如果你绑定到
一个布尔属性，class 名会根据该属性值来添加或移除：

```javascript
App.MyView = Ember.View.extend({
  classNameBindings: ['isUrgent'],
  isUrgent: true
});
```

这会渲染成这样：

```html
<div class="ember-view is-urgent">
```

如果 `isUrgent` 被修改为 `false`，则 `is-urgent` class 会被移除。

默认情况下，布尔属性的值会被“-”变换。你可以用冒号分割，自定义应用到视图的类
名：

```javascript
App.MyView = Ember.View.extend({
  classNameBindings: ['isUrgent:urgent'],
  isUrgent: true
});
```

这会渲染成这份 HTML：

```html
<div class="ember-view urgent">
```

除了值为 `true` 时的自定义 class 名，你也可以指定在值为 `false` 时的 class 名：

```javascript
App.MyView = Ember.View.extend({
  classNameBindings: ['isEnabled:enabled:disabled'],
  isEnabled: false
});
```

这会渲染出这样的 HTML：

```html
<div class="ember-view disabled">
```

你同样也可以只指定属性为 `false` 时添加的 class，只需这样声明 `classNameBindings`：

```javascript
App.MyView = Ember.View.extend({
  classNameBindings: ['isEnabled::disabled'],
  isEnabled: false
});
```

这会渲染这样的 HTML：

```html
<div class="ember-view disabled">
```

如果 `isEnabled` 属性被设置为 `true` ，那么不会添加任何 class：

```html
<div class="ember-view">
```

如果绑定的值是一个字符串，那个值会被直接作为 class 名添加：

```javascript
App.MyView = Ember.View.extend({
  classNameBindings: ['priority'],
  priority: 'highestPriority'
});
```

这会渲染这样的 HTML：

```html
<div class="ember-view highestPriority">
```


### 视图上的属性绑定

你可以用 `attributeBindings` 绑定表示视图 DOM 元素的属性：

```javascript
App.MyView = Ember.View.extend({
  tagName: 'a',
  attributeBindings: ['href'],
  href: "http://emberjs.com"
});
```
