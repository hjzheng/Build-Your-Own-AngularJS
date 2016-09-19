### 销毁 Scope

一个典型的 Angular 应用的生命周期, 页面元素变化是通过不同的视图和数据呈现给用户的. 这意味着 Scope 层级扩大和收缩发生在应用的生命周期中.

在我们的实现中, 可以创建子 Scope, 但是却没有一种机制去删除它们. 当考虑性能时, 一个日益扩大的 Scope 层级是非常不合适的. 因此我们需要一种方式销毁 Scope.

销毁一个 Scope 意味着, Scope 上的所有监听器要被删除并且它自己也要从父 Scope 上的 $$children 属性内删除. 该 Scope 不在需要再被任何地方引用, 它在某个时间会被 JavaScript 垃圾回收器回收.

我们为 Scope 添加一个叫做 $destroy 的方法, 来实现销毁操作:

```js
it('is no longer digested when $destroy has been called', function() {
    var parent = new Scope();
    var child = parent.$new();
    child.aValue = [1, 2, 3];
    child.counter = 0;
    child.$watch(
        function(scope) { return scope.aValue; },
        function(newValue, oldValue, scope) {
            scope.counter++;
        },
        true
    );
    parent.$digest();
    expect(child.counter).toBe(1);
    child.aValue.push(4);
    parent.$digest();
    expect(child.counter).toBe(2);
    child.$destroy();
    child.aValue.push(5);
    parent.$digest();
    expect(child.counter).toBe(2);
});
```

在 $destroy 中, 我们需要当前 Scope 的父 Scope 的引用, 目前还没有, 因此让我们在 $new 中添加一个. 当子 Scope 被创建时, 它的 $parent 属性直接指向父 Scope 的引用.

```js
Scope.prototype.$new = function(isolated, parent) {
  var child;
  parent = parent || this;
  if (isolated) {
    child = new Scope();
    child.$root = parent.$root;
    child.$$asyncQueue = parent.$$asyncQueue;
    child.$$postDigestQueue = parent.$$postDigestQueue;
    child.$$applyAsyncQueue = parent.$$applyAsyncQueue;
  } else {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    child = new ChildScope();
  }
  parent.$$children.push(child);
  child.$$watchers = [];
  child.$$children = [];
  child.$parent = parent;
  return child;
};
```

现在, 来实现 $destroy 方法. 当前 Scope 在其父 Scope 的 $$children 数组找到自己, 并删除自己, 同时确保自己不是 Root Scope 并且有一个父 Scope. 当然, 还要删除注册在自己上的所有监听器:

```js
Scope.prototype.$destroy = function() {
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
