### $apply, $evalAsync 和 $applyAsync 方法需要 digest 整个 Scope 树

正如我们上一小节看到的, $digest 的工作方式, 仅从当前 Scope 向下遍历. 这不是 $apply 的方式. 当我们在 Angular 中调用 $apply 时, 直接从 Root Scope 开始 digest 整个 Scope 层级.
目前, 我们的实现还没有这样做.

```js
it('digests from root on $apply', function() {
  var parent = new Scope();
  var child = parent.$new();
  var child2 = child.$new();
  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  child2.$apply(function(scope) { });
  expect(parent.counter).toBe(1);
});
```
基于当前的实现, 当我们在子 Scope 上调用 $apply 时, 不会触发父 Scope 上的监听器.

为了使测试通过, 首先, 为所有的 Scope 添加一个 Root Scope 的引用. 虽然我们可以通过原型链找到 Root Scope, 但是通过暴露一个 $root 变量访问会直接和方便. 我们在 Root Scope 构造函数添加一个变量.

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
  this.$$phase = null;
}
```

这个单独的变量使 $root 在层级中的每个 Scope 中可用, 这样感谢原型继承链.

接下来, 在 $apply 方法中用在 Root Scope 直接调用 $digest 替换 在当前 Scope 上调用 $digest.

```js
Scope.prototype.$apply = function(expr) {
  try {
    this.$beginPhase('$apply');
    return this.$eval(expr);
  } finally {
    this.$clearPhase();
    this.$root.$digest();
  }
};
```

注意: 因为调用的是 this 上 $eval, 所以对给定函数计算的值是当前 Scope 上的, 而不是 Root Scope 上的. 对 $digest 我们却想要从 Root Scope 往下运行.

事实上, 在 $apply 中 digest 一直都是从 Root Scope 开始的, 使用这种方式处理的原因就是它是在 Angular digest 循环中集成外部代码的首选方案: 如果你不能准确的知道是在那个 Scope 上做的改变, 最安全的方式就是执行所有的 Scope 上的 $digest

很显然, 整个 Angular 应用有且仅有一个 Root Scope, 执行 $apply 方法会引起整个应用中每个 Scope 上的每个监听器执行. 掌握了 $digest 和 $apply 的不同, 当你需要做性能优化时, 有时需要用 $digest 替换 $apply.

通过上面修改, 已经覆盖了 $digest 和 $apply 方法, 还有 $applyAsync 和 $evalAsync 方法需要修改, 先看一个测试:

```js
it('schedules a digest from root on $evalAsync', function(done) {
  var parent = new Scope();
  var child = parent.$new();
  var child2 = child.$new();
  parent.aValue = 'abc';
  parent.counter = 0;

  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
        scope.counter++;
    }
  );

  child2.$evalAsync(function(scope) { });

  setTimeout(function() {
        expect(parent.counter).toBe(1);
        done();
  }, 50);
});
```

这个测试与之前的很相似, 我们检查在 Scope 上调用 $evalAsync 方法, 是否会引起 Root Scope 上的监听器执行.

因为每个 Scope 已经有了 Root Scope 的引用, 所以修改 $evalAsync 很容易.

```js
Scope.prototype.$evalAsync = function(expr) {
  var self = this;
  if (!self.$$phase && !self.$$asyncQueue.length) {
    setTimeout(function() {
      if (self.$$asyncQueue.length) {
        self.$root.$digest();
      }
    }, 0);
  }
  this.$$asyncQueue.push({scope: this, expression: expr});
};
```

在缩短回路优化中, 我们应当一直使用 Root Scope 上的 $$lastDirtyWatch, 无论哪个 Scope 上 $digest 方法被调用.

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var self = this;
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn || function() { },
    last: initWatchVal,
    valueEq: !!valueEq
  };
  this.$$watchers.unshift(watcher);
  this.$root.$$lastDirtyWatch = null;
  return function() {
    var index = self.$$watchers.indexOf(watcher);
    if (index >= 0) {
      self.$$watchers.splice(index, 1);
      self.$root.$$lastDirtyWatch = null;
    }
  };
};
```

同样的 $digest 也应该这样做.

```js
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  this.$root.$$lastDirtyWatch = null;
  this.$beginPhase('$digest');
  if (this.$$applyAsyncId) {
    clearTimeout(this.$$applyAsyncId);
    this.$$ ushApplyAsync();
  }
  do {
    while (this.$$asyncQueue.length) {
      try {
        var asyncTask = this.$$asyncQueue.shift();
        asyncTask.scope.$eval(asyncTask.expression);
      } catch (e) {
        console.error(e);
      }
    }
    dirty = this.$$digestOnce();
    if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
      throw '10 digest iterations reached';
    }
  } while (dirty || this.$$asyncQueue.length);
  this.$clearPhase();
  while (this.$$postDigestQueue.length) {
      try {
        this.$$postDigestQueue.shift()();
      } catch (e) {
        console.error(e);
      }
  }
};
```

最后, 在 $$digestOnce 也做这样处理:

```js
Scope.prototype.$$digestOnce = function() {
  var dirty;
  this.$$everyScope(function(scope) {
    var newValue, oldValue;
    _.forEachRight(scope.$$watchers, function(watcher) {
      try {
        if (watcher) {
          newValue = watcher.watchFn(scope);
          oldValue = watcher.last;
          if (!scope.$$areEqual(newValue, oldValue, watcher.valueEq)) {
             scope.$root.$$lastDirtyWatch = watcher;
             watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
             watcher.listenerFn(newValue,
                 (oldValue === initWatchVal ? newValue : oldValue),
             scope);
             dirty = true;
          } else if (scope.$root.$$lastDirtyWatch === watcher) {
             dirty = false;
            return false;
          }
        }
      } catch (e) {
        console.error(e);
      }
    });
    return dirty !== false;
  });
  return dirty;
};
```