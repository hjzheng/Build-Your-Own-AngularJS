### 当上次 watcher 是干净的, 可以缩短 Digest 回路

在当前实现中, 我们保持完整的遍历所有监听器集合一圈, 直到确认所有 watcher 都是干净的(或者到达TTL), 才会停止.

如果, 在一个 digest 循环中有一个巨大的数量的 watcher 集合, 尽可能的减少执行它的次数是非常重要的, 这也是为什么要对 digest 循环应用特殊优化的原因.

考虑一种情况, 在一个 scope 中, 注册 100 个 watcher, 当我们 digest 时, 第一次遍历这 100 个 watcher 结果都是脏的, 因此, 还要进行第二次遍历, 在第二次循环中, 没有一个 watcher 是脏的, 但是, 在 digest 结束前, 我们仍然执行了 200 次 watcher.

那么, 我们可以做些什么来减少 watcher 的执行次数呢, 只需要记录最近一次结果是脏的 watcher, 然后, 第二次循环的时候, 比较当前执行的 watcher 是否是上次记住的 watcher, 如果是, 说明, 剩余的 watcher 上次的结果都是干净的, 没有必要全部循环完, 直接退出循环就好.

```js
it('ends the digest when the last watch is clean', function() {
  scope.array = _.range(100);
  var watchExecutions = 0;
  _.times(100, function(i) {
    scope.$watch(
      function(scope) {
        watchExecutions++;
        return scope.array[i];
      },
      function(newValue, oldValue, scope) {
      }
    );
  });
  scope.$digest();
  expect(watchExecutions).toBe(200);
  scope.array[0] = 420;
  scope.$digest();
  expect(watchExecutions).toBe(301);
});
```

首先, 我们将一个具有 100 项的数组添加到 scope 上, 接着, 添加 100 个监听器, 每个监听器监控数组中的一项. 我们也添加了一个本地变量, 用来追踪 watcher 总的执行次数.

然后, 运行一次 digest, 初始化所有 watcher, 在这次 digest 中, 每个 watcher 运行了两遍.

此时, 我们修改数组的第一个值, 如果这时缩短回路优化起作用的话, 最终的 watcher 执行次数是 301 而不是 400.

正如上面提到的, 优化是通过记录最近一次结果是脏的 watcher, 让我们在 scope 构造函数中添加 $$lastDirtyWatch 变量来记录吧.

```js
function Scope() {
    this.$$watchers = [];
    this.$$lastDirtyWatch = null;
}
```

当 digest 开始时, 我们将变量 $$lastDirtyWatch 设置为 null.

```js
Scope.prototype.$digest = function() {
    var ttl = 10;
    var dirty;
    this.$$lastDirtyWatch = null;
    do {
        dirty = this.$$digestOnce();
        if (dirty && !(ttl--)) {
            throw '10 digest iterations reached';
        }
    } while (dirty);
 };
```

在 $$digestOnce 中, 当遇到一个返回结果是 dirty 的 watcher, 我们将它赋给 $$lastDirtyWatch 变量.

```js
Scope.prototype.$$digestOnce = function() {
    var self = this;
    var newValue, oldValue, dirty;
        _.forEach(this.$$watchers, function(watcher) {
            newValue = watcher.watchFn(self);
            oldValue = watcher.last;
            if (newValue !== oldValue) {
                self.$$lastDirtyWatch = watcher;
                watcher.last = newValue;
                watcher.listenerFn(newValue,
                    (oldValue === initWatchVal ? newValue : oldValue),
                self);
                dirty = true;
            }
    });
    return dirty;
};
```

仍然在 $$digestOnce 中, 当遇到一个返回结果是干净的 watcher 并且是上一次记录的返回结果是 dirty 的 watcher, 跳出 $digest 循环.

```js
Scope.prototype.$$digestOnce = function() {
    var self = this;
    var newValue, oldValue, dirty;
    _.forEach(this.$$watchers, function(watcher) {
        newValue = watcher.watchFn(self);
        oldValue = watcher.last;
        if (newValue !== oldValue) {
            self.$$lastDirtyWatch = watcher;
            watcher.last = newValue;
            watcher.listenerFn(newValue,
                (oldValue === initWatchVal ? newValue : oldValue),
            self);
            dirty = true;
        } else if (self.$$lastDirtyWatch === watcher) {
            return false; // lodash 的 forEach, 如果返回 false, 会跳出循环.
        }
    });
    return dirty;
};
```

现在优化生效了, 但是还有一种情况需要处理, 在 listener 函数中, 添加监听器.

```js
it('does not end digest so that new watches are not run', function() {
    scope.aValue = 'abc';
    scope.counter = 0;
    scope.$watch(
        function(scope) { return scope.aValue; },
        function(newValue, oldValue, scope) {
            scope.$watch(
                function(scope) { return scope.aValue; },
                function(newValue, oldValue, scope) {
                    scope.counter++;
                }
            );
        }
    );
    scope.$digest();
    expect(scope.counter).toBe(1);
});
```
第二个 watcher 并未执行, 原因是在第二次 digest 循环中, 我们检测到第一个 watcher 就是上次记录的结果返回是脏的 watcher, 跳出循环了. 让我们来修复这个问题:
当我们添加一个新的 watcher 时, 重新设置 $$lastDirtyWatch 为 null, 禁用优化.

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
    var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { },
        last: initWatchVal
    };
    this.$$watchers.push(watcher);
    this.$$lastDirtyWatch = null;
};
```

现在, digest 循环潜在的已经比之前快了很多, 在一个典型的应用中, 这种优化并不能总是生效如上面的例子, 但是在平均情况下, 它已经做的足够好, AngulaJS 团队已经决定引入它.

接下来, 让我们继续关注如何检测数据变化.