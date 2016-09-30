### 向上传播事件

现在我们终于到达 $emit 和 $broadcast 之间的差异这部分了: 事件在 Scope 层级上的传播方向是不同的.

当你发射(emit)一个事件, 该事件会传给当前 Scope 上对应的事件监听器, 然后向上继续传播给所有父级 Scope, 包括 Root Scope 上的事件监听器.

测试必须从早先创建的 forEach 循环中出来, 因为它只包含 $emit:

```js
it('propagates up the scope hierarchy on $emit', function() {
  var parentListener = jasmine.createSpy();
  var scopeListener = jasmine.createSpy();
  parent.$on('someEvent', parentListener);
  scope.$on('someEvent', scopeListener);
  scope.$emit('someEvent');
  expect(scopeListener).toHaveBeenCalled();
  expect(parentListener).toHaveBeenCalled();
});
```

让我们尝试尽可能简单的实现, 在 $emit 方法中遍历 Scope. 我们可以使用第二章中引入的 $parent 属性获取每个 Scope 的父 Scope.

```js
Scope.prototype.$emit = function(eventName) {
    var additionalArgs = _.tail(arguments);
    var scope = this;
    do {
        scope.$$fireEventOnScope(eventName, additionalArgs);
        scope = scope.$parent;
    } while (scope);
};
```

上面的实现几乎没有问题, 除了破坏了之前约定的, 调用 $emit 返回事件对象, 而且还有我们讨论过的为每个 listener 函数传入相同的事件对象参数. 这个需求在不同的 Scope 上都是适用的, 但是下面的测试没有通过:

```js
it('propagates the same event up on $emit', function() {
    var parentListener = jasmine.createSpy();
    var scopeListener = jasmine.createSpy();
    parent.$on('someEvent', parentListener);
    scope.$on('someEvent', scopeListener);
    scope.$emit('someEvent');
    var scopeEvent = scopeListener.calls.mostRecent().args[0];
    var parentEvent = parentListener.calls.mostRecent().args[0];
    expect(scopeEvent).toBe(parentEvent);
});
```

这意味着我们需要撤销之前去除重复代码的工作, 在 $emit 和 $broadcast 中构造事件对象, 然后传入到 $$fireEventOnScope 函数中. 既然已经这样了, 让我们将整个 listener 的参数的构造从 $$fireEventScope 中拉出来.这样的话, 我们不必为每个 Scope 构造事件对象了.

```js
Scope.prototype.$emit = function(eventName) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;
    do {
        scope.$$freEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope);
    return event;
};

Scope.prototype.$broadcast = function(eventName) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$freEventOnScope(eventName, listenerArgs);
    return event;
};

Scope.prototype.$$freEventOnScope = function(eventName, listenerArgs) {
    var listeners = this.$$listeners[eventName] || [];
    var i = 0;
    while (i < listeners.length) {
        if (listeners[i] === null) {
            listeners.splice(i, 1);
        } else {
            listeners[i].apply(null, listenerArgs);
            i++;
        }
    }
};
```

我们在这里引入一点重复代码, 但是还不是太糟糕.
