### 在事件对象中包含当前 Scope 和目标 Scope

此时, 我们的事件对象仅仅包含一个属性: 事件名称, 下一步, 我们准备在事件对象上捆绑更多信息.

如果你对浏览器中的 DOM 事件非常熟悉, 你会知道事件对象会有几个非常有用的属性: target 定义发生事件的 DOM 元素, currentTarget 定义注册事件处理的 DOM 元素.

因为 DOM 事件在 DOM 树上传播, 所以 target 和 currentTarget 有可能是不一样的.

Angular Scope 有一对类似的属性: targetScope 定义事件发生的 Scope, currentScope 定义注册 listener 的 Scope. 并且因为 Scope 事件在 Scope 树上下传播, 所以 targetScope 和 currentScope 也有可能是不同的.

|              |  Event originated in  |  Listener attached in  |
| ------------ | --------------------- | ---------------------- |
| DOM Events   | target                | currentTarget          |
| Scope Events | targetScope           | currentScope           |

开始 targetScope, 它指向相同的 Scope 与注册的 listener 无关.

对于 $emit:

```js
it('attaches targetScope on $emit', function() {
  var scopeListener = jasmine.createSpy();
  var parentListener = jasmine.createSpy();
  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);
  scope.$emit('someEvent');
  expect(scopeListener.calls.mostRecent().args[0].targetScope).toBe(scope);
  expect(parentListener.calls.mostRecent().args[0].targetScope).toBe(scope);
});
```

对于 $broadcast:

```js
it('attaches targetScope on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();
  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  scope.$broadcast('someEvent');
  expect(scopeListener.calls.mostRecent().args[0].targetScope).toBe(scope);
  expect(childListener.calls.mostRecent().args[0].targetScope).toBe(scope);
});
```

为了让测试通过, 我们所要做的是, 在 $emit 和 $broadcast 中, 将 this 作为 targetScope 添加到事件对象中.

```js
Scope.prototype.$emit = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;
    do {
        scope.$$ reEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope);
    return event;
};

Scope.prototype.$broadcast = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$everyScope(function(scope) {
        scope.$$ reEventOnScope(eventName, listenerArgs);
        return true;
    });
    return event;
};
```

相反地, currentScope 的不同, 基于 listener 注册在什么 Scope 上. 它应该指向准确的 Scope. 一种方式是当事件在 Scope 层级中上下传播时, currentScope 指向当前正在传播的 Scope.

在这个用例中, 我们不能对测试使用 Jasmine spy, 因为 spy 只能调用后验证. currentScope 在遍历 Scope 时是一直在改变, 所以我们当 listener 调用时, 需要随时记录它的值.

我们可以用自己的 listener 函数和本地变量搞定:

对于 $emit:

```js
it('attaches currentScope on $emit', function() {
  var currentScopeOnScope, currentScopeOnParent;
  var scopeListener = function(event) {
    currentScopeOnScope = event.currentScope;
  };
  var parentListener = function(event) {
    currentScopeOnParent = event.currentScope;
  };
  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);
  scope.$emit('someEvent');
  expect(currentScopeOnScope).toBe(scope);
  expect(currentScopeOnParent).toBe(parent);
});
```

对于 $broadcast:

```js
it('attaches currentScope on $broadcast', function() {
  var currentScopeOnScope, currentScopeOnChild;
  var scopeListener = function(event) {
    currentScopeOnScope = event.currentScope;
  };
  var childListener = function(event) {
    currentScopeOnChild = event.currentScope;
  };
  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  scope.$broadcast('someEvent');
  expect(currentScopeOnScope).toBe(scope);
  expect(currentScopeOnChild).toBe(child);
});
```

幸运的是实际的实现要比测试写法简单直接的多. 所有我们需要做的是把当前遍历的 Scope 赋值给 currentScope


```js
Scope.prototype.$emit = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;
    do {
        event.currentScope = scope;
        scope.$$fireEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope);
    return event;
};

Scope.prototype.$broadcast = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$everyScope(function(scope) {
        event.currentScope = scope;
        scope.$$fireEventOnScope(eventName, listenerArgs);
        return true;
    });
    return event;
};
```

因为 currentScope 是用来沟通事件传播的当前状态, 事件传播结束后应该被清除掉.

```js
it('sets currentScope to null after propagation on $emit', function() {
  var event;
  var scopeListener = function(evt) {
    event = evt;
  };
  scope.$on('someEvent', scopeListener);
  scope.$emit('someEvent');
  expect(event.currentScope).toBe(null);
});

it('sets currentScope to null after propagation on $broadcast', function() {
  var event;
  var scopeListener = function(evt) {
    event = evt;
  };
  scope.$on('someEvent', scopeListener);
  scope.$broadcast('someEvent');
  expect(event.currentScope).toBe(null);
});
```

通过在 $emit 方法的最后简单设置 currentScope 为 null 就可以完成.

```js
Scope.prototype.$emit = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;
    do {
        event.currentScope = scope;
        scope.$$fireEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope);
    event.currentScope = null;
    return event;
};
```

在 $broadcast 中做同样的处理.

```js
Scope.prototype.$broadcast = function(eventName) {
    var event = {name: eventName, targetScope: this};
    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$everyScope(function(scope) {
        event.currentScope = scope;
        scope.$$fireEventOnScope(eventName, listenerArgs);
        return true;
    });
    event.currentScope = null;
    return event;
};
```

现在事件监听器可以基于在 Scope 层级上的事件的来源于哪里, 在哪里被监听做出决定.
