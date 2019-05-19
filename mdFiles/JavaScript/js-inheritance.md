# JavaScript的继承

JavaScript是个多范式的语言，既可以使用函数式编程的方式开发，也可以使用面向对象的方式开发。而作为面向对象方式中很重要的继承功能，JavaScript使用原型特性来支持。

网上有不少继承的例子，这里讨论一下经典的继承方法和常见的问题。

## 1. 何为继承
继承是面向对象编程中的一个特性。WikiPedia定义是，【继承是一种可以基于一个父类创建出子类的方式，使子类持有类似的实现】。基本上就是，通过继承父类来获取子类，使子类创建出的对象，拥有父类的全部属性和方法。

## 2. JavaScript中如何实现继承
目前比较流行的方式，使用原型链来实现：
``` javascript
// 父类：构造函数
function Animal(name) {
  // 父类：每个实例都有的属性
  this.name = name;
}

// 父类：公共属性
Animal.prototype.sayHi = function() {
  console.log(`${this.name}: Hi~`);
}

// 子类：构造函数
function Cat(name, brand) {
  // 继承步骤1/3： 调用父类构造函数，使父类中实例私有属性初始化
  Animal.call(this, name);
  // 子类：每个实例都有的新属性
  this.brand = brand;
}

// 子类：新的公共属性
let newProperties = {
  meow: {
    value: function() {
      console.log(`${this.name} (${this.brand}): Miao~`);
    }
  }
};

// 继承步骤2/3：连接原型链，使父类的公共属性暴露给子类实例
Cat.prototype = Object.create(Animal.prototype, newProperties);
// 继承步骤3/3：修正constructor属性
Cat.prototype.constructor = Cat;

// 测试一下
let bella = new Animal("Bella");
let kitty = new Cat("Kitty", "Orange Cat");
bella.sayHi();
kitty.sayHi();
kitty.meow();
```

运行结果：
``` bash
$ node test-js-prototype/app.js
Bella: Hi~
Kitty: Hi~
Kitty (Orange Cat): Miao~
```

### 2.1. 延伸问题一
**Object.create()是什么，为什么不直接使用更简单的方式，比如把继承步骤2直接改成：**
``` javascript
// 继承步骤2/3：连接原型链，使父类的公共属性暴露给子类实例
let newPrototype = new Animal();
newPrototype.meow = function() {
  console.log(`${this.name} (${this.brand}): Miao~`);
}
Cat.prototype = newPrototype;
```
如果直接使用一个Animal的对象来作为Cat的prototype属性，那么Animal的构造函数中增加的私有属性，全部都会添加到Cat的prototype中。这样造成的负面影响，就是Cat的实例会有两个name属性，一个在自己实例上，一个在原型链上:
``` javascript
console.log(kitty.hasOwnProperty("name")); // true
console.log(kitty.__proto__.hasOwnProperty("name")); // true
```
实际上，Object.create()函数就是把传入对象作为原型，然后生成一个新的对象，这样就避免了父类构造函数造成的负面影响：
``` javascript
Object.myCreate = function(proto) {
  function F() {};
  F.prototype = proto;
  return new F();
}
```
### 2.2. 延伸问题二
**既然是Animal的构造函数会造成的负面影响，那么直接使用以下方法，岂不是完美解决了问题？**
``` javascript
// 继承步骤2/3：连接原型链，使父类的公共属性暴露给子类实例
let newPrototype = {};
newPrototype.__proto__ = Animal.prototype;
newPrototype.meow = function() {
  console.log(`${this.name} (${this.brand}): Miao~`);
}
Cat.prototype = newPrototype;
```
以上看上去与经典的继承方式是一样的，但是，使用\_\_proto\_\_（或者Object.setPrototypeOf()）来动态更改对象的原型链可能导致性能下降。

浏览器对于原型特性做了很多的优化，在调用实例方式之前尝试提前猜中内存对应的位置。如果对原型链进行了动态修改，不仅会破坏JS引擎的优化，还可能导致有些游览器为了符合JS规范，对JS代码重新编译进行【反优化】操作。

具体的优化细节，可以参照这个文章：[JavaScript engine fundamentals: optimizing prototypes](https://mathiasbynens.be/notes/prototypes)

``` javascript
// This is bad:
foo.__proto__.bar = bar;

// But this is okay:
Foo.prototype.bar = bar;
```

## 3. 对经典的继承方式进行封装
上面的例子讲述了如何实现继承，但是实际使用中，使用这样的接口可能会比较繁琐。很多现有的框架和库都对继承进行了封装，从而提供一个友好的接口。

### 3.1. BackboneJS的封装
例如在BackboneJS中，通常用以下方式来继承：
``` javascript
var AppView = Backbone.View.extend({
 el: "body",
 render: function () {
    this.$el.html("<h1>Hi!</h1>");
    return this;
  },
});
```

我们可以阅读最新的Backbone的源代码，其中的**Backbone.View.extend**函数定义如下：
``` javascript
// 参数protoProps：实例的新属性
// 参数staticProps：新静态属性（类的属性，直接添加到函数上面，不会包含在实例中）
var extend = function(protoProps, staticProps) {
  var parent = this;
  var child;

  // 创建子类
  // 如果新属性中定义了constructor，直接使用；否则的话创建一个空函数作为新的类
  if (protoProps && _.has(protoProps, 'constructor')) {
    child = protoProps.constructor;
  } else {
    // 继承步骤1/3： 调用父类构造函数，使父类中实例私有属性初始化
    child = function(){ return parent.apply(this, arguments); };
  }

  // 把静态属性复制到子类中
  _.extend(child, parent, staticProps);

  // 继承步骤2/3：连接原型链，使父类的公共属性暴露给子类实例
  child.prototype = _.create(parent.prototype, protoProps);
  // 继承步骤3/3：修正constructor属性
  child.prototype.constructor = child;

  // 设置一个便捷的属性，万一以后会使用父类的prototype
  // Backbone中只设置了这个属性，自己却没有使用
  child.__super__ = parent.prototype;

  return child;
};

// 使Backbone的组件，都拥有这个功能
Model.extend = Collection.extend = Router.extend = View.extend = History.extend = extend;
```

由上可以看出，Backbone实现继承的方式，和我们的例子完全一致。但是封装之后，开发者使用起来感觉更容易了。

### 3.2. 其他的经典封装方法
不仅是Backbone，还有很多其他的开发者也对继承进行了封装，增添了一些新的特性（比如规范使用init函数初始化，方法中可以使用\_super()来访问父类的同名方法，等等）。

我之前做的一个项目，继承部分的代码都是这样的：

``` javascript
var Person = Class.extend({
  init: function(isDancing) {
    // 初始化函数
    this.dancing = isDancing;
  },
  dance: function() {
    return this.dancing;
  }
});
 
var Ninja = Person.extend({
  init: function() {
    // 调用父类的初始化函数
    this._super(false);
  },
  dance: function() {
    // 调用父类的同名方法
    return this._super();
  },
  swingSword: function() {
    return true;
  }
});
```

这种方式对于从Java语言转过来的同学非常友好 - 实际上我也是从Java转到前端的:)。

对应的实现原理，和Backbone的实现差不多，如果感兴趣的话，可以看一下John Resig的文章[Simple JavaScript Inheritance](https://johnresig.com/blog/simple-javascript-inheritance/)。

### 3.3 Babel和ES6对继承的封装
对于近几年的新项目，很多开发者都使用了Webpack和Babel来构建。Babel允许开发者直接使用ES6语法来写类和继承。例如下面的React代码：
``` javascript
class App extends React.Component {
  render() {
    return "<h1>Hi!</h1>";
  }
}
```

那么Babel是如何把上述的ES6代码转换成普通浏览器支持的代码呢？在[Babel REPL](https://babeljs.io/repl)在线工具中，左侧输入ES6代码：
``` javascript
class Animal { }

class Cat extends Animal { }
```
右侧就可以看到转换的结果：
``` javascript
function _inherits(subClass, superClass) {
  if (typeof superClass !== "function" && superClass !== null) {
    throw new TypeError("Super expression must either be null or a function");
  }
  // 这里同时完成了：
  // 继承步骤2/3：连接原型链，使父类的公共属性暴露给子类实例
  // 继承步骤3/3：修正constructor属性
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: {
      value: subClass,
      writable: true,
      configurable: true
    }
  });

  // 把静态属性复制到子类中
  // Babel的处理方式与Backbone不一样，可以对比一下
  // 其实这里会对性能有影响的，但是也没有其他的办法
  if (superClass)
    _setPrototypeOf(subClass, superClass);
}

var Animal = function Animal() {
  _classCallCheck(this, Animal);
};

var Cat = function(_Animal) {
  _inherits(Cat, _Animal);
  function Cat() {
    _classCallCheck(this, Cat);

    // _getPrototypeOf(Cat).apply(this, arguments)：
    // 继承步骤1/3： 调用父类构造函数，使父类中实例私有属性初始化
    return _possibleConstructorReturn(this, _getPrototypeOf(Cat).apply(this, arguments));
  }
  return Cat;
}(Animal);
```

所以，Babel也是用的我们在最初就讨论的经典继承实现。

## 参考链接：
* [JavaScript engine fundamentals: optimizing prototypes](https://mathiasbynens.be/notes/prototypes)
* [MDN - Object​.new operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)
* [MDN - Object​.create\(\)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
* [MDN - Object​.set​PrototypeOf\(\)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf)
* [MDN - The performance hazards of Prototype mutation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/The_performance_hazards_of__%5B%5BPrototype%5D%5D_mutation)
* [Simple JavaScript Inheritance](https://johnresig.com/blog/simple-javascript-inheritance/)
* [Providing A Return Value In A JavaScript Constructor](https://www.bennadel.com/blog/2522-providing-a-return-value-in-a-javascript-constructor.htm)
* [StackOverflow - Why does Babel use setPrototypeOf for inheritance when it already does Object.create(superClass.prototype)](https://stackoverflow.com/questions/37926910/why-does-babel-use-setprototypeof-for-inheritance-when-it-already-does-object-cr)
* [StackOverflow - ES6 to ES5: Babel's implementation of class extension](https://stackoverflow.com/questions/48360418/es6-to-es5-babels-implementation-of-class-extension)
* [Babel REPL](https://babeljs.io/repl/)
* [Backbone JS](https://backbonejs.org/)
