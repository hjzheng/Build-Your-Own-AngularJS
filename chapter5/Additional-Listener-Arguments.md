### listener 函数的额外参数

当我们发射(emit)或广播(broadcast)一个事件, 事件名称本身并不足以沟通所发生事情的一切. 为事件添加相关的额外参数是非常常见的. 你只需要在事件名称参数之后添加任意数量的参数就行.

```js
aScope.$emit('eventName', 'and', 'additional', 'arguments');
```

我们需要把这些参数传递给 listener 函数. 它们应该接收到这些参数, 相应的, 额外的参数出现在事件对象参数之后.

```js
it('passes additional arguments to listeners on '+method, function() {
  var listener = jasmine.createSpy();
  scope.$on('someEvent', listener);
  scope[method]('someEvent', 'and', ['additional', 'arguments'], '...');
  expect(listener.calls.mostRecent().args[1]).toEqual('and');
  expect(listener.calls.mostRecent().args[2]).toEqual(['additional', 'arguments']);
  expect(listener.calls.mostRecent().args[3]).toEqual('...');
});
```

在 $emit 和 $broacast 中, 我们抓取传给函数的额外参数, 并将它们作为参数传递给 $$fireEventOnScope. 我们可以使用 LoDash 的 _.tail 函数获得额外的参数.

```js
Scope.prototype.$emit = function(eventName) {
    var additionalArgs = _.tail(arguments);
    this.$$ reEventOnScope(eventName, additionalArgs);
};

Scope.prototype.$broadcast = function(eventName) {
    var additionalArgs = _.tail(arguments);
    this.$$ reEventOnScope(eventName, additionalArgs);
};
```

在 $$fireEventOnScope 中, 我们不能将额外的参数简单的传递给 listener 函数. 那是因为 listener 期望额外的参数不是一个简单的数组而是正常的函数参数. 因此, 我们需要使用 apply 调用 listener 函数.

```js
Scope.prototype.$$fireEventOnScope = function(eventName, additionalArgs) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(additionalArgs);
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener.apply(null, listenerArgs);
    });
};
```
