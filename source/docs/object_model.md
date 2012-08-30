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

### Reopening Classes and Instances

You don't need to define a class all at once. You can reopen a class and
define new properties using the `reopen` method.

```javascript
Person.reopen({
  isPerson: true
});

Person.create().get('isPerson') // true
```

When using `reopen`, you can also override existing methods and
call `this._super`.

```javascript
Person.reopen({
  // override `say` to add an ! at the end
  say: function(thing) {
    this._super(thing + "!");
  }
});
```

As you can see, `reopen` is used to add properties and methods to an instance.
But when you need to create class method or add the properties to the class itself you can use `reopenClass`.

```javascript
Person.reopenClass({
  createMan: function() {
    return Person.create({isMan: true})
  }
});

Person.createMan().get('isMan') // true
```

### Computed Properties (Getters)

Often, you will want a property that is computed based on other
properties. Ember's object model allows you to define computed
properties easily in a normal class definition.

```javascript
Person = Ember.Object.extend({
  // these will be supplied by `create`
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

The `property` method defines the function as a computed property, and
defines its dependencies. Those dependencies will come into play later
when we discuss bindings and observers.

When subclassing a class or creating a new instance, you can override
any computed properties.

### Computed Properties (Setters)

You can also define what Ember should do when setting a computed
property. If you try to set a computed property, it will be invoked with
the key and value you want to set it to.

```javascript
Person = Ember.Object.extend({
  // these will be supplied by `create`
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

Ember will call the computed property for both setters and getters, and
you can check the number of arguments to determine whether it is being called
as a getter or a setter.

### Observers

Ember supports observing any property, including computed properties.
You can set up an observer on an object by using the `addObserver`
method.

```javascript
Person = Ember.Object.extend({
  // these will be supplied by `create`
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
  // deal with the change
});

person.set('firstName', "Brohuda"); // observer will fire
```

Because the `fullName` computed property depends on `firstName`,
updating `firstName` will fire observers on `fullName` as well.

Because observers are so common, Ember provides a way to define
observers inline in class definitions.

```javascript
Person.reopen({
  fullNameChanged: function() {
    // this is an inline version of .addObserver
  }.observes('fullName')
});
```

You can define inline observers by using the `Ember.observer` method if you
are using Ember without prototype extensions:

```javascript
Person.reopen({
  fullNameChanged: Ember.observer(function() {
    // this is an inline version of .addObserver
  }, 'fullName')
});
```

#### Changes in Arrays

Often, you may have a computed property that relies on all of the items in an
array to determine its value. For example, you may want to count all of the
todo items in a controller to determine how many of them are completed.

Here's what that computed property might look like:

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

Note here that the dependent key (`todos.@each.isDone`) contains the special
key `@each`. This instructs Ember.js to update bindings and fire observers for
this computed property when one of the following four events occurs:

1. The `isDone` property of any of the objects in the `todos` array changes.
2. An item is added to the `todos` array.
3. An item is removed from the `todos` array.
4. The `todos` property of the controller is changed to a different array.

In the example above, the `remaining` count is `1`:

```javascript
App.todosController.get('remaining');
// 1
```

If we change the todo's `isDone` property, the `remaining` property is updated
automatically:

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


### Bindings

A binding creates a link between two properties such that when one changes, the
other one is updated to the new value automatically. Bindings can connect
properties on the same object, or across two different objects. Unlike most other
frameworks that include some sort of binding implementation, bindings in
Ember.js can be used with any object, not just between views and models.

The easiest way to create a two-way binding is by creating a new property
with the string `Binding` at the end, then specifying a path from the global scope:

```javascript
App.wife = Ember.Object.create({
  householdIncome: 80000
});

App.husband = Ember.Object.create({
  householdIncomeBinding: 'App.wife.householdIncome'
});

App.husband.get('householdIncome'); // 80000

// Someone gets raise.
App.husband.set('householdIncome', 90000);
App.wife.get('householdIncome'); // 90000
```

Note that bindings don't update immediately. Ember waits until all of your
application code has finished running before synchronizing changes, so you can
change a bound property as many times as you'd like without worrying about the
overhead of syncing bindings when values are transient.

#### One-Way Bindings

A one-way binding only propagates changes in one direction. Usually, one-way
bindings are just a performance optimization and you can safely use
the more concise two-way binding syntax (as, of course, two-way bindings are
de facto one-way bindings if you only ever change one side).

```javascript
App.user = Ember.Object.create({
  fullName: "Kara Gates"
});

App.userView = Ember.View.create({
  userNameBinding: Ember.Binding.oneWay('App.user.fullName')
});

// Changing the name of the user object changes
// the value on the view.
App.user.set('fullName', "Krang Gates");
// App.userView.userName will become "Krang Gates"

// ...but changes to the view don't make it back to
// the object.
App.userView.set('userName', "Truckasaurus Gates");
App.user.get('fullName'); // "Krang Gates"
```

### What Do I Use When?

Sometimes new users are confused about when to use computed properties,
bindings and observers. Here are some guidelines to help:

1. Use *computed properties* to build a new property by synthesizing other
properties. Computed properties should not contain application behavior, and
should generally not cause any side-effects when called. Except in rare cases,
multiple calls to the same computed property should always return the same
value (unless the properties it depends on have changed, of course.)

2. *Observers* should contain behavior that reacts to changes in another
property. Observers are especially useful when you need to perform some
behavior after a binding has finished synchronizing.

3. *Bindings* are most often used to ensure objects in two different layers
are always in sync. For example, you bind your views to your controller using
Handlebars. You may often bind between two objects in the same layer. For
example, you might have an `App.selectedContactController` that binds to the
`selectedContact` property of `App.contactsController`.
