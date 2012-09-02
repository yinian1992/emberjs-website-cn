## 创建命名空间

每个  Ember 应用应该有一个 `Ember.Application` 的实例。这个对象充当
你的应用中所有其它类和实例的全局可访问的命名空间。此外，它在页面上设
置事件监听器，这样当用户与用户界面交互时你的视图会接收到事件（之后会
了解）。

这里是一个应用的例子：

```javascript
window.App = Ember.Application.create();
```

你可以给命名空间起任何名字，只是需要以一个大写字母开头来让绑定能找到
它。

如果你向一个现存的站点中嵌入一个 Ember 应用，你可以通过提供一个
`rootElement` 属性来为指定元素设置好事件监听器：

```javascript
window.App = Ember.Application.create({
  rootElement: '#sidebar'
});
```
