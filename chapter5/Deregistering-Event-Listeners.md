### 注销事件监听器

在我们进入 $emit 和 $broadcast 之间的区别之前, 让我们来看一个重要的共同需求: 不仅可以注册事件监听器, 而且可以注销事件监听器.

注销事件监听器的原理和注销 watcher 是一样的, 回到第一章的实现: The registration function returns a deregistration function.  一旦注销函数被调用, 事件监听器不会接收任何事件:

```js
it('can be deregistered '+method, function() {
  var listener = jasmine.createSpy();
  var deregister = scope.$on('someEvent', listener);
  deregister();
  scope[method]('someEvent');
  expect(listener).not.toHaveBeenCalled();
});
```

一个简单的删除实现就如同在 $watch 中做的一样 - 只要从集合中 splice 监听函数:

```js
Scope.prototype.$on = function(eventName, listener) {
    var listeners = this.$$listeners[eventName];
    if (!listeners) {
        this.$$listeners[eventName] = listeners = [];
    }
    listeners.push(listener);
    return function() {
        var index = listeners.indexOf(listener);
        if (index >= 0) {
            listeners.splice(index, 1);
        }
    };
};
```

这里有一个特殊的情况, 我们必须要小心, 无论如何: 在监听器触发时删除自己是非常常见的, 举例当我们只调用 listener 一次. 这种删除发生在循环 listeners 数组时, 会导致跳过一个 listener.

```js
it('does not skip the next listener when removed on '+method, function() {
  var deregister;
  var listener = function() {
    deregister();
  };
  var nextListener = jasmine.createSpy();
  deregister = scope.$on('someEvent', listener);
  scope.$on('someEvent', nextListener);
  scope[method]('someEvent');
  expect(nextListener).toHaveBeenCalled();
});
```

意思就是你不能简单的直接过去删除掉 listener. 我们能做的是标识出 listener 已经删除了, null 就很好.

```js
Scope.prototype.$on = function(eventName, listener) {
    var listeners = this.$$listeners[eventName];
    if (!listeners) {
        this.$$listeners[eventName] = listeners = [];
    }
    listeners.push(listener);
    return function() {
        var index = listeners.indexOf(listener);
        if (index >= 0) {
            listeners[index] = null;
        }
    };
};
```

然后, 在循环 listener 时, 我们检测 listener 如果为 null, splices listener. 我们要做就是从使用 _.forEach 切换到手动 while 循环.

```js
Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(additionalArgs);
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
    return event;
};
```
