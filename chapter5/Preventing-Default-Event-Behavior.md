### 阻止事件默认行为

除了 stopPropagation 之外, DOM 事件也可以被用另一种方式取消, 这种方式就是阻止事件的默认行为. DOM 事件有一个叫做 preventDefault 的函数就是阻止默认事件的.
它的目的是阻止浏览器原生事件, 但是仍然允许注册的监听事件执行. 举例, 当 preventDefault 在超链接的 click 事件触发时被调用, 浏览器不会执行超链接的跳转行为, 但是注册的 click 事件仍然会被执行.

Scope 的事件也有一个 preventDefault 函数. 然而, Scope 事件没有内置的默认行为, 所以调用 preventDefault 函数的作用很小. 它做了一件事, 在事件对象上设置一个叫做 defaultPrevented 的布尔标识.
这个标识没有改变 Scope 事件系统的行为, 但是它被用于自定义指令中一旦事件完成传播决定是否触发默认行为. 当广播 location 事件时, AngularJS 的 $locationService 就是这样做的.

因此, 我们需要一个测试: 当一个 listener 函数, 在 event 对象上调用 preventDefault(), 它的 defaultPrevented 标识会被设置. 这个行为在 $emit 和 $broadcast 中是一致的.

```js
it('is sets defaultPrevented when preventDefault called on '+method, function() {
    var listener = function(event) {
        event.preventDefault();
    };
    scope.$on('someEvent', listener);
    var event = scope[method]('someEvent');
    expect(event.defaultPrevented).toBe(true);
});
```

这里的实现与 stopPropagation 中我们所做的非常相似: 有一个函数设置布尔标识并将它添加到事件对象上. 不同的是这次布尔标识也被添加到事件对象上, 并且这次不会基于该布尔值做任何决定.

对于 $emit:

```js
Scope.prototype.$emit = function(eventName) {
    var propagationStopped = false;
    var event = {
        name: eventName,
        targetScope: this,
        stopPropagation: function() {
            propagationStopped = true;
        },
        preventDefault: function() {
            event.defaultPrevented = true;
        }
    };

    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;

    do {
        event.currentScope = scope;
        scope.$$freEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope && !propagationStopped);

    return event;
};
```

对于 $broadcast:

```js
Scope.prototype.$broadcast = function(eventName) {
    var event = {
        name: eventName,
        targetScope: this,
        preventDefault: function() {
            event.defaultPrevented = true;
        }
    };

    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$everyScope(function(scope) {
        event.currentScope = scope;
        scope.$$freEventOnScope(eventName, listenerArgs);
        return true;
    });
    return event;
};
```
