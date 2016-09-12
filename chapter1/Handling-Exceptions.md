### 处理异常

我们的脏检查的实现与 Angular 的非常类似了, 但是, 它相当的脆弱, 主要是因为我们没有对异常处理花费太多精力.

如果一个异常发生在 watch 函数中, 当前的实现将会停止工作. 而 Angular 的实现却要比它健壮的多, Angular 会在 digest 中捕获和记录抛出的异常, 并恢复之前的操作.

在监听器中, 有两个产生异常的地方: 一个是 watch 函数, 一个是 listener 函数, 在下面的测试用例中, 我们期望异常被记录下一个 watch 继续执行.

```js
it('catches exceptions in watch functions and continues', function() {
  scope.aValue = 'abc';
  scope.counter = 0;
  scope.$watch(
    function(scope) { throw 'Error'; },
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
it('catches exceptions in listener functions and continues', function() {
  scope.aValue = 'abc';
  scope.counter = 0;
  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      throw 'Error';
    }
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

我们定义了两个 watch, 第一个 watch 抛出异常, 我们检查第二个 watch 正常执行.

为了使上面的测试通过, 我们必须改变 $$digestOnce 函数, 对每个 watch 的执行进行 try catch 处理:

```js
Scope.prototype.$$digestOnce = function() {
  var self = this;
  var newValue, oldValue, dirty;
  _.forEach(this.$$watchers, function(watcher) {
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
现在, 我们的 digest 循环健壮多了, 当遇到异常的时候.
