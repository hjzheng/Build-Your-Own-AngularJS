### 向下传播事件

基本上 $broadcast 就是 $emit 的镜像: 它也是调用当前 Scope 以及当前 Scope 直接的, 间接的, 无论是否隔离的子 Scope 上的 listener.

```js
it('propagates down the scope hierarchy on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();
  var isolatedChildListener = jasmine.createSpy();
  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  isolatedChild.$on('someEvent', isolatedChildListener);
  scope.$broadcast('someEvent');
  expect(scopeListener).toHaveBeenCalled();
  expect(childListener).toHaveBeenCalled();
  expect(isolatedChildListener).toHaveBeenCalled();
});
```

为了测试的完整性, 在 $broadcast 传播事件时, 让我们也确保一个相同的事件对象传给所有的 listener 函数.

```js
it('propagates the same event down on $broadcast', function() {
  var scopeListener = jasmine.createSpy();
  var childListener = jasmine.createSpy();
  scope.$on('someEvent', scopeListener);
  child.$on('someEvent', childListener);
  scope.$broadcast('someEvent');
  var scopeEvent = scopeListener.calls.mostRecent().args[0];
  var childEvent = childListener.calls.mostRecent().args[0];
  expect(scopeEvent).toBe(childEvent);
});
```

在 $broadcast 中遍历 Scope 不像在 $emit 中那么简单明确, 因为不是一条直接向下的路径, 反而所以的 Scope 分散在一个树状结构. 我需要做的是遍历这个树.
更准确的说, 我们需要使用与在 $$digestOnce 中一样的深度优先算法去遍历树. 我们可以重用第二章引入的 $$everyScope 方法.

```js
Scope.prototype.$broadcast = function(eventName) {
    var event = {name: eventName};
    var listenerArgs = [event].concat(_.tail(arguments));
    this.$$everyScope(function(scope) {
        scope.$$fireEventOnScope(eventName, listenerArgs);
        return true;
    });
    return event;
};
```

现在很显然, 为什么 broadcast 一个事件可能会比 emit 一个事件昂贵: emit 在一个直线上向上传播事件, 通常 Scope 的层级也不会很深. 但是 broadcast 却要遍历一个树. 如果在 Root Scope 上 broadcast, 将要访问整个应用的每一个 Scope.
