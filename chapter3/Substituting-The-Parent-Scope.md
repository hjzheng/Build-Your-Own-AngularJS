### 替换父 Scope

在一些情况中, 通过传递其他 Scope 作为新 Scope 的父 Scope, 同时仍然要维护原 Scope 上原型继承链是非常有用的:

```js
it('can take some other scope as the parent', function() {
  var prototypeParent = new Scope();
  var hierarchyParent = new Scope();
  var child = prototypeParent.$new(false, hierarchyParent);
  prototypeParent.a = 42;
  expect(child.a).toBe(42);
  child.counter = 0;
  child.$watch(function(scope) {
    scope.counter++;
  });
  prototypeParent.$digest();
  expect(child.counter).toBe(0);
  hierarchyParent.$digest();
  expect(child.counter).toBe(2);
});
```
我们用构造函数创建两个父 Scope, 然后创建一个子 Scope, 其中一个父 Scope 是新 Scope 原型链上的 Scope, 另一个则是 Scope 树层级上的父 Scope.

我们测试在原型继承上的父子 Scope 之间可以和之前一样工作, 但是我们测试了再原型继承的父 Scope 运行 digest, 没有触发子 Scope 上的监听器运行.

相反的, 在层级上的父 Scope 上却成功的触发子 Scope 上的监听器运行.

我们为 $new 方法, 引入第二个可选参数, 默认是当前 Scope, 我们会使用该 Scope 的 children 存储新的 Scope.

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
    return child;
};
```

这个特性引进了原型继承和层级继承之间的细微的区别, 可能在大部分情况下, 它的价值不大, 不值得花费精力区分两种 Scope 继承的细微不同. 但是当我们实现指令的 transclusion 特性时, 它会非常的有用.
