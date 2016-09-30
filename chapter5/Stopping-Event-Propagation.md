### 阻止事件传递

DOM 事件的另一个非常常用的特性, 就是阻止事件传播特性. DOM 事件对象有一个叫做 stopPropagation 的函数就是这个目的, 它经常被用于在多个层级 DOM 上 handler, 又不想触发特定事件的所有 handler 的情景.

Scope 事件也有一个 stopPropagation 方法, 但是只能当 emit 事件时使用. broadcast 时事件不能被阻止.

这里的意思是当你 emit 一个事件, 所有注册 listener 函数中的一个, 阻止事件的传播, 父 Scope 上 listener 将不会看到这些事件.

```js
it('does not propagate to parents when stopped', function() {
  var scopeListener = function(event) {
    event.stopPropagation();
  };
  var parentListener = jasmine.createSpy();
  scope.$on('someEvent', scopeListener);
  parent.$on('someEvent', parentListener);
  scope.$emit('someEvent');
  expect(parentListener).not.toHaveBeenCalled();
});
```

因此事件不会向父 Scope 传播, 但是, 关键是事件仍然会传递给当前 Scope 中所有剩余的 listener, 只是阻止向父 Scope 穿破的事件.

```js
it('is received by listeners on current scope after being stopped', function() {
  var listener1 = function(event) {
    event.stopPropagation();
  };
  var listener2 = jasmine.createSpy();
  scope.$on('someEvent', listener1);
  scope.$on('someEvent', listener2);
  scope.$emit('someEvent');
  expect(listener2).toHaveBeenCalled();
});
```

第一件事就是我们需要一个表示是否调用 stopPropagation 方法布尔标志. 我们可以在 $emit 中引入该标志. 然后就是需要 stopPropagation 函数自己并将它添加到事件对象里. 最后在 do ... while 循环中进入父 Scope 之前检查布尔标志的状态.

```js
Scope.prototype.$emit = function(eventName) {
    var propagationStopped = false;
    var event = {
        name: eventName,
        targetScope: this,
        stopPropagation: function() {
            propagationStopped = true;
        }
    };
    var listenerArgs = [event].concat(_.tail(arguments));
    var scope = this;
    do {
        event.currentScope = scope;
        scope.$$fireEventOnScope(eventName, listenerArgs);
        scope = scope.$parent;
    } while (scope && !propagationStopped);
    return event;
};
```
