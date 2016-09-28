### $emit 和 $broadcast 的基础

现在, 我们拥有注册的事件监听器, 我们可以触发事件去使用它们, 正如我们讨论过的: 有两个方法可以触发事件: $emit 和 $broadcast.

这两个函数的基本功能是当你用一个事件名称作为参数调用它们的时候, 它们会调用所有用该事件名注册的所有的事件监听器. 相应地, 当然它们不会调用其他事件名称注册的事件监听器.

```js
it('calls the listeners of the matching event on $emit', function() {
    var listener1 = jasmine.createSpy();
    var listener2 = jasmine.createSpy();
    scope.$on('someEvent', listener1);
    scope.$on('someOtherEvent', listener2);
    scope.$emit('someEvent');
    expect(listener1).toHaveBeenCalled();
    expect(listener2).not.toHaveBeenCalled();
});

it('calls the listeners of the matching event on $broadcast', function() {
    var listener1 = jasmine.createSpy();
    var listener2 = jasmine.createSpy();
    scope.$on('someEvent', listener1);
    scope.$on('someOtherEvent', listener2);
    scope.$broadcast('someEvent');
    expect(listener1).toHaveBeenCalled();
    expect(listener2).not.toHaveBeenCalled();
});
```

为了使测试通过, 我们引入 $emit 和 $broadcast 方法: 目前, 它们的行为是一致的. 找到对应事件名称的事件监听器, 成功调用这些监听器:

```js
Scope.prototype.$emit = function(eventName) {
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener();
    });
};

Scope.prototype.$broadcast = function(eventName) {
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener();
    });
};
```
