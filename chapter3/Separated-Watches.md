### 隔开监听器

我们已经看到我们可以在子 Scope 上添加监听器, 因为子 Scope 继承了包括 $watch 和 $digest 在内的所有父 Scope 的方法, 但是这些监听器存放在哪里呢? 被那些 Scope
执行呢?

在当前的实现中, 所有的监听器实际上是被储存在 Root Scope 上. 这是因为我们将 $$watchers 数组定义在 Root Scope 的构造函数中. 当任何子 Scope 访问 $$watchers 数组时, 它们得到是 Root Scope 上的一个 copy.

这种实现有一个严重的后果: 无论我们在哪个 Scope 上调用 $digest 方法, 都会在 Scope 的层级中执行所有的将监听器, 这是因为只有一个监听器数组. 这并不是我们想要的结果.

我们真正想要的是, 在 Scope 上调用 $digest 方法时, 该 Scope 上和它的所有子 Scope 上的监听器被执行, 而不是它的父 scope 或其它 scope 上的子 scope 的监听器.

```js
it('does not digest its parent(s)', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = 'abc';
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```

这个测试是失败的, 因为当我们调用 child.$digest() 时, 我们执行了父 Scope 上的监听器.

让我们来修复它, 窍门就是为每个子 Scope 添加一个 $$watcher 数组:

```js
Scope.prototype.$new = function() {
  var ChildScope = function() { };
  ChildScope.prototype = this;
  var child = new ChildScope();
  child.$$watchers = [];
  return child;
};
```

你或许已经注意到了, 我们在这使用了属性遮蔽. 每个子 Scope 上的 $$watchers 数组遮蔽了父 Scope 的. 在层级里的每个 Scope 都有自己的 watcher 数组. 当我们调用 Scope 上的 $digest, scope 上的监听器会被执行.
