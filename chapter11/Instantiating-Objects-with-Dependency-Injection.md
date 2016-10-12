### Instantiating Objects with Dependency Injection

我们决定在本章为 injector 添加一种能力: 不仅可以注入普通的函数也可以注入构造函数:

当你拥有一个构造函数并想将它实例化成对象, 同时也要注入依赖. 你可以使用 injector.instantiate, 它可以处理带有显示依赖标注的构造函数:

```js
it('instantiates an annotated constructor function', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);
    var injector = createInjector(['myModule']);
    function Type(one, two) {
        this.result =  one + two;
    }
    Type.$inject = ['a', 'b'];
    var instance = injector.instantiate(Type);
    expect(instance.result).toBe(3);
});
```

当然也可以处理数组包裹的风格的标注:

```js
it('instantiates an array-annotated constructor function', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);
    var injector = createInjector(['myModule']);
    function Type(one, two) {
        this.result = one + two;
    }
    var instance = injector.instantiate(['a', 'b', Type]);
    expect(instance.result).toBe(3);
});
```

最后一种, 从构造函数源码中抽取依赖名称:

```js
it('instantiates a non-annotated constructor function', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);
    var injector = createInjector(['myModule']);
    function Type(a, b) {
        this.result = a + b;
    }
    var instance = injector.instantiate(Type);
    expect(instance.result).toBe(3);
});
```

让我们在 injector 中引入 instantiate 方法. 它指向一个本地方法, 我们暂时引入到这:

```js
return {
    has: function(key) {
        return cache.hasOwnProperty(key);
    },
    get: function(key) {
        return cache[key];
    },
    annotate: annotate,
    invoke: invoke,
    instantiate: instantiate
};
```

一个非常简单的 instantiate 的实现仅仅是创建一个新对象, 将新对象作为构造函数的上下文, 用 invoke 函数执行构造函数, 然后返回新对象:

```js
function instantiate(Type) {
    var instance = {};
    invoke(Type, instance);
    return instance;
}
```

这样确实可以是我们现有的测试通过. 但是使用构造函数有一个非常重要的行为: 当你使用 new 构造对象时, 你也基于构造函数的原型链设置了对象的原型链. 在 injector.instantiate 中, 我们需要遵循这种行为.

举例, 如果我们实例化一个在 prototype 上增加行为的构造函数, 实例化的对象也可以通过继承使用增加的行为.

```js
it('uses the prototype of the constructor when instantiating', function() {
    function BaseType() { }
    BaseType.prototype.getValue = _.constant(42);
    function Type() { this.v = this.getValue(); }
    Type.prototype = BaseType.prototype;
    var module = window.angular.module('myModule', []);
    var injector = createInjector(['myModule']);
    var instance = injector.instantiate(Type);
    expect(instance.v).toBe(42);
});
```

设置原型链, 我们可以使用 ES5 的 Object.create 方法代替一个简单的字面量构建一个对象. 我们还要记得去掉函数外的数组, 因为有可能使用数组风格的依赖标注.

```js
function instantiate(Type) {
    var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
    var instance = Object.create(UnwrappedType.prototype);
    invoke(Type, instance);
    return instance;
}
```

最终, 就像 injector.invoke 支持本地参数一样, injector. instantiate 也要支持:

```js
it('supports locals when instantiating', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);
  function Type(a, b) {
    this.result = a + b;
  }
  var instance = injector.instantiate(Type, {b: 3});
  expect(instance.result).toBe(4);
});
```

我们只需要把 local 参数作为的第三个参数传给 invoke.

```js
function instantiate(Type, locals) {
    var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
    var instance = Object.create(UnwrappedType.prototype);
    invoke(Type, instance, locals);
    return instance;
}
```
