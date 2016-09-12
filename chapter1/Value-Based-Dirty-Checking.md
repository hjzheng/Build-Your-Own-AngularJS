### 基于值得脏检查

目前, 我们已经使用严格等于 === 比较旧值和新值. 在大多数情况下, 当检测所有基本类型的变化, 对象或数组变化成一个新的时候, 是没有问题的.
但是, 还有另一种 Angular 可以检测的变化, 那就是检测对象和数组内部的变化. 也就是说, 你不仅能检测引用的变化, 也能检测值得变化.

通过对 $watch 方法提供第三个可选的布尔参数启用这种脏检查, 当这个参数是 true 时, 使用基于值的检查, 让我们添加一个测试用例吧:

```js
it('compares based on value if enabled', function() {
  scope.aValue = [1, 2, 3];
  scope.counter = 0;
  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    },
    true
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.aValue.push(4);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

当 scope.aValue 数组变化时, counter 加 1, 当我们往数组里添加东西时, 我们期望它作为一个变化被通知.

让我们来重新定义 $watch 方法:

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
    var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn || function() { },
        valueEq: !!valueEq,
        last: initWatchVal
    };
    this.$$watchers.push(watcher);
    this.$$lastDirtyWatch = null;
};
```

我们所做是为 watcher 添加一个标志, 通过 !! 操作符将其转成布尔值.

基于值得脏检查表明如果旧值或新值是对象或者数组的时, 我们必须遍历它们中的一切. 如果在两个值中有任何不同, watcher 就是脏的, 如果存在对象和数组嵌套, 它们将会被递归的进行值比较.

Angular 拥有自己的等于检查函数, 但是我们将会使用 LoDash 提供的来代替. 让我们重新定义一个新的比较函数.

```js
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  if (valueEq) {
    return _.isEqual(newValue, oldValue);
  } else {
    return newValue === oldValue;
  }
};
```

注意值得变化, 我们也需要改变旧值得存储方式, 不能只存当前值的引用, 因为任何对该值的改变都会被应用到该值的引用, 我们通过 $$areEqual 比较同一值的两个引用, 结果永远都是相等的. 基于这个原因, 我们需要写一个深 copy 函数, copy 该值, 并存储它.

让我们来修改 $digestOnce 方法吧:

```js
Scope.prototype.$$digestOnce = function() {
  var self = this;
  var newValue, oldValue, dirty;
  _.forEach(this.$$watchers, function(watcher) {
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
  });
  return dirty;
};
```

基于值的检查明显比基于引用的检查引入了更多的操作, 有时候更多. 遍历嵌套数据结构需要花费时间, 保存一个深 copy 需要消耗更多内存, 这也是为什么 Angular 默认不是基于值的检查.