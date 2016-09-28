### 注册事件监听器: $on

你需要注册一个事件, 才能得到事件通知. AngularJS 中的注册完成是通过调用 Scope 对象上的 $on 方法. 这个方法需要两个参数: 事件名称和当事件发生时被调用的监听器函数.

通过 $on 注册的监听器可以接收发射和广播事件. 实际上除了事件名称, 对事件的接收没有任何方式的限制.

$on 方法应该做些什么呢? 它应该在某个地方存放监听器函数, 当事件发生时, 可以找到对应的监听器函数. 对于如何存放, 我们将一个对象放在 $$listeners 属性上, 对象的键是事件名称, 值是一个存放为特定事件注册的监听器函数的数组. 所以, 我们可以测试用 $on 注册的函数是否被相应存储:

```js
it('allows registering listeners', function() {
    var listener1 = function() { };
    var listener2 = function() { };
    var listener3 = function() { };
    scope.$on('someEvent', listener1);
    scope.$on('someEvent', listener2);
    scope.$on('someOtherEvent', listener3);
    expect(scope.$$listeners).toEqual({
        someEvent: [listener1, listener2],
        someOtherEvent: [listener3]
    });
});
```

我们需要将 $$listeners 对象添加到 Scope 上. 让我们在 constructor 设置一下:

```js
function Scope() {
    this.$$watchers = [];
    this.$$lastDirtyWatch = null;
    this.$$asyncQueue = [];
    this.$$applyAsyncQueue = [];
    this.$$applyAsyncId = null;
    this.$$postDigestQueue = [];
    this.$root = this;
    this.$$children = [];
    this.$$listeners = {};
    this.$$phase = null;
}
```

$on 函数应该检查在给定事件上是否已经存在一个 listener 集合, 如果没有, 初始化一个. 有的话, 在集合上添加一个新的 listener 函数.

```js
Scope.prototype.$on = function(eventName, listener) {
    var listeners = this.$$listeners[eventName];
    if (!listeners) {
        this.$$listeners[eventName] = listeners = [];
    }
    listeners.push(listener);
};
```

因为在 Scope 层级上的每个监听器位置是相同的, 当前的 $$listeners 的实现确实有一个小问题: 在整个 Scope 层级上的所有监听器会被添加到同一个 $$listeners 集合中. 相反的, 我们需要为每个 Scope 添加单独的 $$listeners 集合.

```js
it('registers different listeners for every scope', function() {
    var listener1 = function() { };
    var listener2 = function() { };
    var listener3 = function() { };
    scope.$on('someEvent', listener1);
    child.$on('someEvent', listener2);
    isolatedChild.$on('someEvent', listener3);
    expect(scope.$$listeners).toEqual({someEvent: [listener1]});
    expect(child.$$listeners).toEqual({someEvent: [listener2]});
    expect(isolatedChild.$$listeners).toEqual({someEvent: [listener3]});
});
```

This test fails because both scope and child actually have a reference to the same $$listeners
collection and isolatedChild doesn’t have one at all. We need to tweak the child scope constructor to explicitly give each new child scope its own $$listeners collection. For a non-isolated scope it will shadow the one in its parent. This is exactly the same solution as we used for
$$watchers in Chapter 2:

这个测试失败了, 因为 Scope 和子 Scope 实际上共同拥有相同 $$listeners 集合的引用, 并且隔离的子 Scope 完全没有. 我们需要调整一下子 Scope 的构造函数: 显示地给每个新的子 Scope 添加一个它自己的 $$listeners 集合. 对于非隔离 scope, $$listeners 属性会遮蔽它父 Scope 的. 这与第二章中我们为 $$watcher 使用的解决方案是一样的.

```js
Scope.prototype.$new = function(isolated, parent) {
    var child;
    parent = parent || this;
    if (isolated) {
        child = new Scope();
        child.$root = parent.$root;
        child.$$asyncQueue = parent.$$asyncQueue;
        child.$$postDigestQueue = parent.$$postDigestQueue;
        child.$$applyAsyncQueue = this.$$applyAsyncQueue;
    } else {
        var ChildScope = function() { };
        ChildScope.prototype = this;
        child = new ChildScope();
    }
    parent.$$children.push(child);
    child.$$watchers = [];
    child.$$listeners = {};
    child.$$children = [];
    child.$parent = parent;
    return child;
};
```
