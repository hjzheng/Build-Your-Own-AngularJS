### 返回事件对象

$emit 和 $broadcast 都有的一个额外的特性就是它们都返回自己构建的事件对象, 因此事件的发起者在事件完成传播后可以检测事件的状态.

```js
it('returns the event object on '+method, function() {
  var returnedEvent = scope[method]('someEvent');
  expect(returnedEvent).toBeDe ned();
  expect(returnedEvent.name).toEqual('someEvent');
});
```

该实现非常简单, 仅仅返回事件对象:

```js
Scope.prototype.$emit = function(eventName) {
    var additionalArgs = _.tail(arguments);
    return this.$$fireEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$broadcast = function(eventName) {
    var additionalArgs = _.tail(arguments);
    return this.$$fireEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(additionalArgs);
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener.apply(null, listenerArgs);
    });
    return event;
};
```
