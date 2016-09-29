### 事件对象

当前, 我们调用 listener 函数时没有带任何参数, 但是 AngularJS 不是这样做的. 我们应该如何做: 为 listener 函数传入一个事件对象.

这个事件对象其实就是一个携带与事件相关的信息和行为的常规 JavaScript 对象. 我们会在该事件对象上添加一些属性, 但首先, 为事件对象添加一个记录事件的名称的 name 属性.

```js
it('passes an event object with a name to listeners on '+method, function() {
  var listener = jasmine.createSpy();
  scope.$on('someEvent', listener);
  scope[method]('someEvent');
  expect(listener).toHaveBeenCalled();
  expect(listener.calls.mostRecent().args[0].name).toEqual('someEvent');
});
```

事件对象的另一个重要方面是完全相同的事件对象作为参数被传入到每个 listener 函数中. 应用开发者可以向事件对象上添加额外的属性在 listener 函数之间进行通信.

```js
it('passes the same event object to each listener on '+method, function() {
  var listener1 = jasmine.createSpy();
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);
  scope[method]('someEvent');
  var event1 = listener1.calls.mostRecent().args[0];
  var event2 = listener2.calls.mostRecent().args[0];
  expect(event1).toBe(event2);
});
```

我们可以在 $$fireEventOnScope 函数中构造这个事件对象, 并将它作为参数传入 listener 函数:

```js
Scope.prototype.$$fireEventOnScope = function(eventName) {
    var event = {name: eventName};
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener(event);
    });
};
```

