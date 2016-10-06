### 广播 Scope 删除

有时候知道 Scope 被删除是很有用的. 一个典型的应用场景就是在指令中你可能设置了一些 DOM 事件 listener 和其他引用, 当指令的元素销毁时, 需要被清理. 解决方案是在指令的 Scope 中监听一个叫做 $destroy 的事件.

$destroy 事件来自哪里呢? 当一个 Scope 被删除的时候, 也就是当某人调用 Scope 的 $destroy 方法时, 我们应当触发它.

```js
it('fres $destroy when destroyed', function() {
    var listener = jasmine.createSpy();
    scope.$on('$destroy', listener);
    scope.$destroy();
    expect(listener).toHaveBeenCalled();
});
```

当一个 Scope 被删除的时候, 它的所有子 Scope 也被删除了, 它们的 listener 也应该接收到 $destroy 事件.

```js
it('fres $destroy on children destroyed', function() {
    var listener = jasmine.createSpy();
    child.$on('$destroy', listener);
    scope.$destroy();
    expect(listener).toHaveBeenCalled();
});
```

我们如何让它工作呢? 正好有一个函数可以满足在 Scope 和它的子 Scope 中触发事件的需求: $broadcast. 我们需要在 $destroy 函数中使用 $broadcast 广播 $destroy 事件.

```js
Scope.prototype.$destroy = function() {
    this.$broadcast('$destroy');
    if (this.$parent) {
        var siblings = this.$parent.$$children;
        var indexOfThis = siblings.indexOf(this);
        if (indexOfThis >= 0) {
            siblings.splice(indexOfThis, 1);
        }
    }
    this.$$watchers = null;
};
```
