### 销毁监听器

当你注册一个监听器的时候, 大部分的时候, 你希望它与 scope 一样一直存在, 不希望显示的移除它, 但是有一些情况, 我们想要从 scope 中删除它. 因此我们需要一个删除操作:

Angular 的实现方式非常聪明: Angular 中的 $watch 函数 有一个返回值, 它是一个函数, 当被调用的时候, 会销毁注册的函数.

```js
it('allows destroying a $watch with a removal function', function() {
  scope.aValue = 'abc';
  scope.counter = 0;
  var destroyWatch = scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.aValue = 'def';
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.aValue = 'ghi';
  destroyWatch();
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

为了实现它, 我们需要 $watch 返回一个可以将监听器从 $$watchers 数组中删除的函数.

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
    var self = this;
    var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn,
        valueEq: !!valueEq,
        last: initWatchVal
    };
    self.$$watchers.push(watcher);
    this.$$lastDirtyWatch = null;
    return function() {
        var index = self.$$watchers.indexOf(watcher);
        if (index >= 0) {
            self.$$watchers.splice(index, 1);
        }
    };
};
```

在一个健壮实现之前, 还有一些边角情况需要我们处理. 我们所要做的就是注意, 在 digest 中删除 watch 的情况.

首先, 监听器在 watch 或 listener 函数中删除自己, 不应当影响其他监听器.

```js
it('allows destroying a $watch during digest', function() {
  scope.aValue = 'abc';
  var watchCalls = [];
  scope.$watch(
    function(scope) {
      watchCalls.push('first');
      return scope.aValue;
    }
  );
  var destroyWatch = scope.$watch(
    function(scope) {
      watchCalls.push('second');
      destroyWatch();
    }
  );
  scope.$watch(
    function(scope) {
      watchCalls.push('third');
      return scope.aValue;
    }
  );
  scope.$digest();
  expect(watchCalls).toEqual(['first', 'second', 'third', 'first', 'third']);
});
```

在这个测试中, 我们有三个 watch, 中间的 watch 删除了他自己, 当第一次被调用的时候, 只剩下第一个和第三个.

我们需要做两个调整:

第一, 添加一个 watch 的时候, 使用 unshift 代替 push

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
  this.$$lastDirtyWatch = null;
  return function() {
    var index = self.$$watchers.indexOf(watcher);
    if (index >= 0) {
      self.$$watchers.splice(index, 1);
    }
  };
};
```

第二, 使用 _.forEachRight 替换 _.forEach

```js
Scope.prototype.$$digestOnce = function() {
  var self = this;
  var newValue, oldValue, dirty;
  _.forEachRight(this.$$watchers, function(watcher) {
    try {
      newValue = watcher.watchFn(self);
      oldValue = watcher.last;
      if (!self.$$areEqual(newValue, oldValue, watcher.valueEq)) {
        self.$$lastDirtyWatch = watcher;
        watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
        watcher.listenerFn(newValue,
          (oldValue === initWatchVal ? newValue : oldValue),
          self);
        dirty = true;
      } else if (self.$$lastDirtyWatch === watcher) {
        return false;
      }
    } catch (e) {
      console.error(e);
    }
  });
  return dirty;
};
```

下一个情况, 在一个 watch 函数中删除另一个监听器, 观察下面的测试用例:

```js
it('allows a $watch to destroy another during digest', function() {
  scope.aValue = 'abc';
  scope.counter = 0;
  scope.$watch(
    function(scope) {
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {
      destroyWatch();
    }
  );
  var destroyWatch = scope.$watch(
    function(scope) { },
    function(newValue, oldValue, scope) { }
  );
  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

这个测试失败了, 罪魁祸首是我们的缩短回路优化和从右向左遍历引起的:

- 第一个 watcher 的 watch 被执行, 它是脏的, 被存储在 $$lastDirtyWatch, 它的 listener 被执行, 销毁第二个 watcher, 数组变短, 它成了第二个.
- 第一个 watcher 又被执行了一遍, 这次它是干净的, 于是跳出循环, 第三个 watcher 始终不执行.

这种情况, 不要进行优化:

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var self = this;
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn,
    valueEq: !!valueEq,
    last: initWatchVal
  };
  self.$$watchers.unshift(watcher);
  this.$$lastDirtyWatch = null;
  return function() {
      var index = self.$$watchers.indexOf(watcher);
      if (index >= 0) {
        self.$$watchers.splice(index, 1);
        self.$$lastDirtyWatch = null;
      }
  };
};
```

最后一种情况, 一个 watch 中删除多个监听器

```js
it('allows destroying several $watches during digest', function() {
  scope.aValue = 'abc';
  scope.counter = 0;
  var destroyWatch1 = scope.$watch(
    function(scope) {
      destroyWatch1();
      destroyWatch2();
    }
  );
  var destroyWatch2 = scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(0);
});
```

第一个 watch 函数, 不仅销毁了自己的监听器, 还销毁了第二个监听器, 但是第二个监听器仍然会被执行, 此时我们不希望出现一个异常.

```js
Scope.prototype.$$digestOnce = function() {
  var self = this;
  var newValue, oldValue, dirty;
  _.forEachRight(this.$$watchers, function(watcher) {
    try {
        if (watcher) {
            newValue = watcher.watchFn(self);
            oldValue = watcher.last;
            if (!self.$$areEqual(newValue, oldValue, watcher.valueEq)) {
                self.$$lastDirtyWatch = watcher;
                watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
                watcher.listenerFn(newValue,
                    (oldValue === initWatchVal ? newValue : oldValue),
                self);
                dirty = true;
            } else if (self.$$lastDirtyWatch === watcher) {
                return false;
            }
        }
    } catch (e) {
      console.error(e);
    }
  });
  return dirty;
};
```
现在, 我们终于可以在 digest 中安全的删除监听器了.