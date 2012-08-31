## Ember 对象模型

Ember 加强了简单 JavaScript 对象模型来支持绑定和观察者，同样也支持一个
更强大的基于混合类型的代码共享方法。

从最基本的开始，用 `Ember.Object` 上的 `extend` 方法创建一个新 Ember
类。

```javascript
Person = Ember.Object.extend({
  say: function(thing) {
    alert(thing);
 }
});
```

一旦你已经建立了一个新类，你可以用 `create` 方法来创建一个该类的新实例。
任何在类中定义的属性会在实例中可用。

```javascript
var person = Person.create();
person.say("Hello") // alerts "Hello"
```

创建实例时，你也可以通过传递一个对象来向实例添加额外的属性。

```javascript
var tom = Person.create({
  name: "Tom Dale",

  helloWorld: function() {
    this.say("Hi my name is " + this.get('name'));
  }
});

tom.helloWorld() // alerts "Hi my name is Tom Dale"
```

由于 Ember 的绑定和观察者支持，你总是会用 `get` 方法来访问属性而用
`set` 方法来设置属性。

当创建一个对象的新实例，你也可以覆盖类中定义的任何属性或方法。例如，
在本例中，你可以覆盖 `Person` 类中的 `say` 方法。

```javascript
var yehuda = Person.create({
  name: "Yehuda Katz",

  say: function(thing) {
    var name = this.get('name');

    this._super(name + " says: " + thing);
  }
});
```

你可以用对象上的 `_super` 方法（`super` 是 JavaScript 保留字）来调用
你覆盖掉的原始方法。

### 继承

你也可以用 `extend` 创建你所创建的类的子类。事实上，上面我们调用
`Ember.Object` 上的 `extend` 方法创建新类时，我们就是在创建一个
`Ember.Object` 的子类。

```javascript
var LoudPerson = Person.extend({
  say: function(thing) {
    this._super(thing.toUpperCase());
  }
});
```

继承时，你可以用 `this._super` 来调用你正在覆盖的方法。

### 重新打开类和实例

你不需要一开始就定义好完整的类。你可以用 `reopen` 方法重新打开一
个类来定义新的属性。

```javascript
Person.reopen({
  isPerson: true
});

Person.create().get('isPerson') // true
```

使用 `reopen` 时，你也可以覆盖现有的方法并调用 `this._super`。

```javascript
Person.reopen({
  // 覆盖 `say` 为在结尾加上一个叹号
  say: function(thing) {
    this._super(thing + "!");
  }
});
```

如你所见，`reopen` 用于向实例添加属性和方法。但当你需要创建类
方法或向类本身添加属性，你可以使用 `reopenClass`。

```javascript
Person.reopenClass({
  createMan: function() {
    return Person.create({isMan: true})
  }
});

Person.createMan().get('isMan') // true
```

### 计算属性（Getter）

很多时候，你会需要一个基于其它属性计算出的属性。Ember 的对象模
型允许你在常规的类定义中轻松定义计算属性。

```javascript
Person = Ember.Object.extend({
  // 这会被 `create` 支持
  firstName: null,
  lastName: null,

  fullName: function() {
    var firstName = this.get('firstName');
    var lastName = this.get('lastName');

   return firstName + ' ' + lastName;
  }.property('firstName', 'lastName')
});

var tom = Person.create({
  firstName: "Tom",
  lastName: "Dale"
});

tom.get('fullName') // "Tom Dale"
```

`property` 方法把函数定义为一个计算属性，也定义了依赖。这些
依赖会在之后介绍绑定和观察者时提到。

当继承一个类或创建一个新实例时，你可以覆盖任何计算属性。

### 计算属性（Setter）

你同样可以定义当给一个计算属性赋值时 Ember 应该做什么。如果
你试图给一个计算属性赋值，它会被调用，并传入你想要赋的键和
值。

```javascript
Person = Ember.Object.extend({
  // 这会被 `create` 支持
  firstName: null,
  lastName: null,

  fullName: function(key, value) {
    // getter
    if (arguments.length === 1) {
      var firstName = this.get('firstName');
      var lastName = this.get('lastName');

      return firstName + ' ' + lastName;

    // setter
    } else {
      var name = value.split(" ");

      this.set('firstName', name[0]);
      this.set('lastName', name[1]);

      return value;
    }
  }.property('firstName', 'lastName')
});

var person = Person.create();
person.set('fullName', "Peter Wagenet");
person.get('firstName') // Peter
person.get('lastName') // Wagenet
```

Ember 会为 setter 和 getter 都调用计算属性。并且你可以检查
参数的数量来决定要作为 getter 还是 setter 来调用。

### 观察者

Ember 支持观察任何属性，包括计算属性。你可以用 `addObserver`
方法来给对象设置一个观察者。

```javascript
Person = Ember.Object.extend({
  // 这会被 `create` 支持
  firstName: null,
  lastName: null,

  fullName: function() {
    var firstName = this.get('firstName');
    var lastName = this.get('lastName');

    return firstName + ' ' + lastName;
  }.property('firstName', 'lastName')
});

var person = Person.create({
  firstName: "Yehuda",
  lastName: "Katz"
});

person.addObserver('fullName', function() {
  // 处理变更
});

person.set('firstName', "Brohuda"); // 这会触发观察者
```

因为计算属性 `fullName` 依赖 `firstName`，更新 `firstName` 也会
触发 `fullName` 上的观察者。

由于观察者如此常用，Ember 提供了类声明中内联的观察者定义方式。

```javascript
Person.reopen({
  fullNameChanged: function() {
    // this is an inline version of .addObserver
  }.observes('fullName')
});
```

如果你正在使用没有原型扩展的 Ember，你可以用 `Ember.observer`
方法来定义内联的观察者：

```javascript
Person.reopen({
  fullNameChanged: Ember.observer(function() {
    // this is an inline version of .addObserver
  }, 'fullName')
});
```

#### 数组中的变化

很多情况，你可能有一个依赖于某个数组中的所有元素来确定值的计算
属性。例如，你可能想对某个控制器中所有的 todo 元素进行计数来决
定其中有多少是完成的。

那个计算属性看起来会是这样：

```javascript
App.todosController = Ember.Object.create({
  todos: [
    Ember.Object.create({ isDone: false })
  ],

  remaining: function() {
    var todos = this.get('todos');
    return todos.filterProperty('isDone', false).get('length');
  }.property('todos.@each.isDone')
});
```

注意依赖键（`todos.@each.isDone`）包含了特殊的关键字 `@each` 。
这让 Ember.js 在下列四种情况中任意一种发生时更新绑定并激活观察
者：

1. `todos` 数组中任意对象的 `isDown` 属性发生变化。
2. 一个元素被添加到 `todos` 数组。
3. `todos` 数组中的一个元素被删除
4. 控制器的 `todos` 属性换成另一个数组

在上面的例子中， `remaining` 计数为 `1` ：

```javascript
App.todosController.get('remaining');
// 1
```
如果我们修改了 todo 的 `isDown` 属性， `remaining` 属性会自动
更新：

```javascript
var todos = App.todosController.get('todos');
var todo = todos.objectAt(0);
todo.set('isDone', true);

App.todosController.get('remaining');
// 0

todo = Ember.Object.create({ isDone: false });
todos.pushObject(todo);

App.todosController.get('remaining');
// 1
```


### 绑定

绑定创建了两个属性之间的联系，如此当其中的一个变更时，另一个也
会自动更新到新的值。绑定可以关联同一个对象上的属性，也可以关联
两个不同对象间的属性。不像大多数其它的框架包含某种程度的绑定实
现，在 Ember.js 中绑定可以在任意对象上使用，而不仅仅局限在视图
与模型之间。

创建一个双向绑定的最简单的方法就是创建一个名称以 `Binding` 结
尾的新属性，之后指定一个相对全局作用域的路径。

```javascript
App.wife = Ember.Object.create({
  householdIncome: 80000
});

App.husband = Ember.Object.create({
  householdIncomeBinding: 'App.wife.householdIncome'
});

App.husband.get('householdIncome'); // 80000

// 某些人涨了。
App.husband.set('householdIncome', 90000);
App.wife.get('householdIncome'); // 90000
```

注意绑定不是立即更新的。Ember 在所有应用代码执行完毕后才会同
步变更，所以你无限次修改一个限定的属性，而不用担心瞬时值的绑
定同步带来的开销。

#### 单向绑定

单向绑定只会在一个方向上传递。通常，使用单向绑定只是为了优化
性能，你可以安全地使用更简洁的双向绑定语法（当然同样地，如果
你总是改变单面，双向绑定实际上就是单向绑定）。

```javascript
App.user = Ember.Object.create({
  fullName: "Kara Gates"
});

App.userView = Ember.View.create({
  userNameBinding: Ember.Binding.oneWay('App.user.fullName')
});

// 修改 user 对象的 name 也会修改视图中的值。
App.user.set('fullName', "Krang Gates");
// App.userView.userName 会变成 "Krang Gates"

// ...但是视图中的变更不会返回到对象上。
App.userView.set('userName', "Truckasaurus Gates");
App.user.get('fullName'); // "Krang Gates"
```

### 术业有专攻

有时新用户会困扰何时使用计算属性、绑定和观察者。下面给出的指
导会有所裨益：

1. 用 *计算属性* 合成其它属性来构成一个新属性。计算属性不应该
包含应用行为，并且几乎不应该在调用时导致任何副作用。除了在极
少情况下，多次调用相同的计算属性应该始终返回相同值（当然除非
属性的依赖已经更改。）

2. *观察者* 应该包含对另一属性中变更反应的行为。当你需要在绑定
完成同步后执行一些行为时，观察者特别的有用。

3. *绑定* 是保证两个不同层的对象始终保持同步的最常用手段。例
如，你用 Handlebars 绑定视图到控制器上。你可能会经常绑定同一
个层的两个对象。例如，你可能会有一个绑定到
`App.contactsController` 的 `selectedContact` 属性的
`App.selectedContactController`。
