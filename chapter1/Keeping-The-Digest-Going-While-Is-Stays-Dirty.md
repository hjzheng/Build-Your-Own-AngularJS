### 当数据为 Dirty 时, 持续 Digest

Digest 核心实现都在这里了, 但是离完成还很远, 举例, 有一种相当典型的场景还不支持: listener 函数本身也可以改变 scope 上的属性, 如果这种情况发生的话, 并且另一个监听器监控这个属性, 即使这个属性改变, 也不会在同一个 digest 中获得属性的变化.

```js
it('triggers chained watchers in the same digest', function() {
    scope.name = 'Jane';
    scope.$watch(
        function(scope) { return scope.nameUpper; },
        function(newValue, oldValue, scope) {
            if (newValue) {
                scope.initial = newValue.substring(0, 1) + '.';
            }
    });
    scope.$watch(
        function(scope) { return scope.name; },
        function(newValue, oldValue, scope) {
            if (newValue) {
                scope.nameUpper = newValue.toUpperCase();
            }
    });
    scope.$digest();
    expect(scope.initial).toBe('J.');
    scope.name = 'Bob';
    scope.$digest();
    expect(scope.initial).toBe('B.');
});
```
在该 scope 上我们有两个监听器: 一个监控 nameUpper 属性并在对应的 listener 为 initial 属性赋值, 另一个则监控 name 属性并在对应的 listener 为 nameUpper 赋值,
我们期望 scope 上的 name 属性发生变化时, 在 digest 中 nameUpper 和 initial 属性也被更新.

然而, 我们需要修改 digest, 确保它遍历所有的 watch 直到监控的值停止变化.

首先, 重命名当前的 $digest 函数为 $$digestOnce, 修改其逻辑, 以确保所有监听器(watcher)执行一次, 并返回一个布尔值, 来确定是否有变化.

```js
Scope.prototype.$$digestOnce = function() {
    var self = this;
    var newValue, oldValue, dirty;
    _.forEach(this.$$watchers, function(watcher) {
        newValue = watcher.watchFn(self);
        oldValue = watcher.last;
        if (newValue !== oldValue) {
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
然后, 让我们来重新定义 $digest, 如下所示:

```js
Scope.prototype.$digest = function() {
  var dirty;
  do {
    dirty = this.$$digestOnce();
  } while (dirty);
};
```
现在 $digest 至少会运行所有监听器一次, 在第一次结束, 有监控的值发生变化, 标记为dirty, 所有的监听器在运行第二次, 直到没有监控的值发生变化.

我们现在可以对 AngularJS 的监听函数有另外一个重要结论：它们可能在单次 digest 里面被执行多次。这也就是为什么人们经常说，监听器应当是幂等的：一个监听器应当没有边界效应，或者边界效应只应当发生有限次。比如说，假设一个监控函数触发了一个 Ajax 请求，无法保证你的应用程序发了多少个请求。