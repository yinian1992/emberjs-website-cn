## Ember 枚举量 API

### 什么是枚举量？

在 Ember 中，枚举量是任何包含若干子对象的对象，并且润徐你在那些子对象上使用枚举量接口。
最基本的枚举量就是内置的 JavaScript 数组。

例如，所有的枚举量支持标准的 `forEach` 方法：

```javascript
[1,2,3].forEach(function(item) {
  console.log(item);
});
```

一般地，枚举量方法，诸如 `forEach` ，接受额外的第二个参数，即回调函数中的 `this` 值：

```javascript
var array = [1,2,3];

array.forEach(function(item) {
  console.log(item, this.indexOf(item));
}, array)
```

还有其它理由，你会发现当使用另一个枚举量方法作为 `forEach` 回调是很有用的：

```javascript
var array = [1,2,3];

array.forEach(array.removeObject, array);
```

注意：第二个参数用于解决 JavaScript 中把方法中的 `this` 设置为 `window` 的限制。

### Ember 中的可枚举量

通常，表示成列表的 Ember 对象实现了枚举量接口。例如：

 * *Array*: Ember 用枚举量接口扩展了原生的 JavaScript 数组。
 * *ArrayProxy*: 一个包裹原生数组并为视图层添加额外功能的构造。
 * *Set*: 一个可以快速会带是否包含某对象的对象。

### 枚举量接口

#### 参数

枚举量方法的回调接受三个参数：

 * *item*: 当前迭代的项目。
 * *index*: 从 0 开始计的整数。
 * *self*: 枚举量本身。

#### 枚举

要枚举一个可枚举对象的所有值，使用 `forEach` 方法：

```javascript
enumerable.forEach(function(item, index, self) {
  console.log(item);
});
```

要在一个可枚举对象上的每个元素调用某个方法，使用 `invoke` 方法：

```javascript
Person = Ember.Object.extend({
  sayHello: function() {
    console.log("Hello from " + this.get('name'));
  }
});

var people = [
  Person.create({ name: "Juan" }),
  Person.create({ name: "Charles" }),
  Person.create({ name: "Majd" })
]

people.invoke('sayHello');

// Hello from Juan
// Hello from Charles
// Hello from Majd
```

#### 首和尾

你可以通过获取 `firstObject` 或 `lastObject` 来获取可枚举量的第一个和最后一
个对象。

```javascript
[1,2,3].get('firstObject') // 1
[1,2,3].get('lastObject')  // 3
```

#### 转换成数组

这非常简单。要把一个可枚举量转换成数组，只需调用它的 `toArray` 方法。


#### 变换

你可以用 `map` 方法把一个可枚举量变换成一个派生的数组：

```javascript
['Goodbye', 'cruel', 'world'].map(function(item, index, self) {
  return item + "!";
});

// 返回 ["Goodbye!", "cruel!", "world!"]
```

#### 在每个对象上设置和获取值

`forEach` 和 `map` 的一个十分常用的用法是获取（或设置）每个元素上的属性。你
可以用 `getEach` 和 `setEach` 方法来完成这些任务。

```javascript
var arr = [Ember.Object.create(), Ember.Object.create()];

// 我们现在有了一个包含两个 Ember.Objects 的数组

arr.setEach('name', 'unknown');
arr.getEach('name') // ['unknown', 'unknown']
```

#### 过滤

另一个要在可枚举量上经常执行的任务是把可枚举量作为输入，并返回一个基于某些条
件过滤后的数组。

对任意个过滤，使用（你可能已经猜到了） `filter` 方法。 `filter` 方法在回调为
`true` 时，把值加入最终的数组中，而 `false` 和 `undefined` 不会加入该值。

```javascript
var arr = [1,2,3,4,5];

arr.filter(function(item, index, self) {
  if (item < 4) { return true; }
})

// 返回 [1,2,3]
```

当处理一个 Ember 对象的集合时，你会经常要基于某些属性的值来过滤一组对象。那
么 `filterProperty` 方法提供了捷径。

```javascript
Todo = Ember.Object.extend({
  title: null,
  isDone: false
});

todos = [
  Todo.create({ title: 'Write code', isDone: true }),
  Todo.create({ title: 'Go to sleep' })
];

todos.filterProperty('isDone', true);

// 返回一个只包含第一个元素的数组
```

如果你要只返回第一个匹配的值，而不是一个包含所有匹配值的数组，你可以使用
`find` 和 `findProperty` ，其机制类似 `filter` 和 `filterProperty` ，只不过
仅返回一个项目。

#### 聚合信息（all 或 any）

如果你要查明可枚举量中的每个项目是否匹配某个条件，你可以使用 `every` 方法：

```javascript
Person = Ember.Object.extend({
  name: null,
  isHappy: false
});

var people = [
  Person.create({ name: 'Yehuda', isHappy: true }),
  Person.create({ name: 'Majd', isHappy: false })
];

people.every(function(person, index, self) {
  if(person.get('isHappy')) { return true; }
});

// 返回 false
```

如果你要查明在可枚举量中是否至少一个项目匹配某个条件，你可以用 `some` 方法：

```javascript
people.some(function(person, index, self) {
  if(person.get('isHappy')) { return true; }
});

// 返回 true
```

如同过滤方法， `every` 和 `some` 方法也有类似的 `everyProperty` 和
`someProperty` 方法。

```javascript
people.everyProperty('isHappy', true) // false
people.someProperty('isHappy', true)  // true
```
