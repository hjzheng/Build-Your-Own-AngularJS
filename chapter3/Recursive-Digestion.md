### 递归 Digest

在上一章, 我们讨论过, 调用 $digest 不应该执行父层级上的监听器, 但是应该执行子层级上的监听器, 这是有意义的, 因为一些子层级上的监听器是可以通过原型链监控当前 scope 上的属性的.

因为每个 Scope 上都有单独的监听器数组, 当我们在父 Scope 上调用 digest 方法时, 子 Scope 不会调用 digest. 我们必须修正 $digest 的行为, 不仅要调用自己的 digest, 也要运行子 scope 的 digest.

第一个问题是, 当前的 Scope 根本不知道自己有没有子 Scope,或者有哪些子 Scope. 我们需要每个 Scope 记录它有哪些子的 Scope.

```js
it('keeps a record of its children', function() {
    var parent = new Scope();
    var child1 = parent.$new();
    var child2 = parent.$new();
    var child2_1 = child2.$new();

    expect(parent.$$children.length).toBe(2);
    expect(parent.$$children[0]).toBe(child1);
    expect(parent.$$children[1]).toBe(child2);
    expect(child1.$$children.length).toBe(0);
    expect(child2.$$children.length).toBe(1);
    expect(child2.$$children[0]).toBe(child2_1);
});
```

我们需要在 Root Scope 构造函数初始化 $$children 数组:

```js
function Scope() {
    this.$$watchers = [];
    this.$$lastDirtyWatch = null;
    this.$$asyncQueue = [];
    this.$$applyAsyncQueue = [];
    this.$$applyAsyncId = null;
    this.$$postDigestQueue = [];
    this.$$children = [];
    this.$$phase = null;
}
```
然后, 我们需要为新创建的 Scope 添加一个新的 $$children 数组, 我们也需要将它们的子 Scope 也放入它们自己的 $$children 数组中. 看看 $new 方法的变化

```js
Scope.prototype.$new = function() {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    var child = new ChildScope();
    this.$$children.push(child);
    child.$$watchers = [];
    child.$$children = [];
    return child;
};
```
我们现在已经将所有子 Scope 记录起来了. 我们可以开始讨论如何 digest 它们. 我们希望在父 Scope 上调用 $digest 方法, 也可以执行到子 Scope 中注册的监听器.

```js
it('digests its children', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  parent.$digest();
  expect(child.aValueWas).toBe('abc');
});
```
为了使上面的测试通过, 我们需要修改 $$digestOnce, 遍历层级运行监听器, 为了处理起来更容易, 添加一个帮助函数 $$everyScope

```js
Scope.prototype.$$everyScope = function(fn) {
  if (fn(this)) {
    return this.$$children.every(function(child) {
      return child.$$everyScope(fn);
    });
  } else {
    return false;
  }
};
```

该函数在当前的 scope 上调用 fn 一次, 并且递归的调用当前 Scope 的子 Scope.

我们现在可以在 $$digestOnce 中使用这个函数为整个操作做一个外层循环.

```js
Scope.prototype.$$digestOnce = function() {
    var dirty;
    var continueLoop = true;
    var self = this;
    this.$$everyScope(function(scope) {
        var newValue, oldValue;
        _.forEachRight(scope.$$watchers, function(watcher) {
            try {
                if (watcher) {
                    newValue = watcher.watchFn(scope);
                    oldValue = watcher.last;
                    if (!scope.$$areEqual(newValue, oldValue, watcher.valueEq)) {
                        self.$$lastDirtyWatch = watcher;
                        watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
                        watcher.listenerFn(newValue,
                            (oldValue === initWatchVal ? newValue : oldValue),
                        scope);
                        dirty = true;
                    } else if (self.$$lastDirtyWatch === watcher) {
                        continueLoop = false;
                        return false;
                    }
                }
            } catch (e) {
                console.error(e);
            }
        });
        return continueLoop;
    });
    return dirty;
};
```

现在 $$digestOnce 函数可以遍历整个层级并且通过返回一个布尔值区分在层级中任意位置任意监听器是否是脏.

循环内部遍历 Scope 的层级, 直到所有 Scope 被访问或者缩短回路优化生效. 缩短回路优化使用 continueLoop 变量追踪. 如果它是 false, 则跳出
循环和 $$digestOnce 函数.

注意, watch 函数必须被传入正确的 scope, 在循环内部用使用特定的 scope 变量替换 this 引用, 确保正确运行.

注意, $$lastDirtyWatch 属性, 我们一直使用的是最顶部 scope 的, 缩短回路优化需要保证 scope 层级中的所有的监听器是一起的. 如果我们在在当前的 Scope 上设置 $$lastDirtyWatch, 它会遮蔽父 Scope 上的属性.





