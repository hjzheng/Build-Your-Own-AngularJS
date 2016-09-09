### 初始化监听的值

比较 watch 函数返回的值和之前存储在 last 属性上的值在大部分时间工作都不会有问题, 但是首次执行 watch 函数都做了什么, 那时, 我们并未设置 last 值, last 是 undefined. 当 watch 函数首次返回的值也是 undefined 时, 是不会工作的.
在这种情况下 listener 函数也应当被调用. 但是当前的实现凡是并未考虑初始化值为 undefined 的情况.

```js
it('calls listener when watch value is first undefined', function() {
  scope.counter = 0;
  scope.$watch(
    function(scope) { return scope.someValue; },
    function(newValue, oldValue, scope) { scope.counter++; }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```
这种情况下 listener 函数也应该被调用, 我们需要保证 last 属性初始化的值必须唯一, 确保它与 watch 函数返回任何值都不同, 包括 undefined.

函数很适合这个目的, 因为 JavaScript 函数是引用值, 不会和任何值相同除过他们自己. 让我们在 scope.js 最上面引入一个函数作为值.

```js
function initWatchVal() { }
```

现在, 我们可以用这个函数来初始化 last 属性

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn,
    last: initWatchVal
  };
  this.$$watchers.push(watcher);
};
```
这种方式, 新的 watch 总会使它们的 listener 函数调用, 无论 watch 函数返回什么.

但是, initWatchVal 函数会作为 oldValue 参数传入 listener 函数, 我们不想将 initWatchVal 函数泄露到 scope.js 之外, 因此 我们需要用新值替换旧值.

```js
it('calls listener with new value as old value the  rst time', function() {
  scope.someValue = 123;
  var oldValueGiven;
  scope.$watch(
    function(scope) { return scope.someValue; },
    function(newValue, oldValue, scope) { oldValueGiven = oldValue; }
  );
  scope.$digest();
  expect(oldValueGiven).toBe(123);
});
```

在 $digset 中, 我们调用 listener, 我们只需要检查旧值是否是 initWatchVal 函数, 并将它替换掉.

```js
Scope.prototype.$digest = function() {
  var self = this;
  var newValue, oldValue;
  _.forEach(this.$$watchers, function(watcher) {
    newValue = watcher.watchFn(self);
    oldValue = watcher.last;
    if (newValue !== oldValue) {
        watcher.last = newValue;
        watcher.listenerFn(newValue,
            (oldValue === initWatchVal ? newValue : oldValue),
        self);
    }
  });
};
```