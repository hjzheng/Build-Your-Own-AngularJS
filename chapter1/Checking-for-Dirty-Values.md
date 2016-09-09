### 脏值检查

根据上面的描述, 监听器 (watcher) 的 watch 函数应当返回我们关心并可以改变的数据. 通常该数据存在于 scope 中. 我们将当前 scope 作为参数去调用 watch 函数, 这样可以很方便的在 watch 函数中访问 Scope 上的数据, 如下所示:

```js
function(scope) {
  return scope.firstName;
}
```

这是 watch 函数的通常写法: 直接返回 scope 上属性.

让我们添加一个测试用例, 用来检查 scope 确实作为 watch 函数的参数:

```js
it("calls the watch function with the scope as the argument", function() {
    var watchFn = jasmine.createSpy();
    var listenerFn = function() { };
    scope.$watch(watchFn, listenerFn);

    scope.$digest();
    expect(watchFn).toHaveBeenCalledWith(scope);
});
```

这次, 我们为 watch 函数创建一个 Spy 用来检查 watch 函数的调用, 最简单通过该测试的方式是, 如下面那样修改 `$degist` 函数:

```js
Scope.prototype.$digest = function() {
    var self = this;
    _.forEach(this.$$watchers, function(watcher) {
        watcher.watchFn(self);
        watcher.listenerFn();
    });
};
```
当然这与我们之后的写法完全不同, `$digest` 函数的任务是调用 watch 函数获取其返回值再与 watch 函数上次的返回值进行比较. 如果值不同, 该监听器就是脏的, 并且它的 listener 函数应该被调用.
让我们继续添加一个测试用例:

```js
it("calls the listener function when the watched value changes", function() {
    scope.someValue = 'a';
    scope.counter = 0;
    scope.$watch(
        function(scope) { return scope.someValue; },
        function(newValue, oldValue, scope) { scope.counter++; }
    );

    expect(scope.counter).toBe(0);
    scope.$digest();
    expect(scope.counter).toBe(1);
    scope.$digest();
    expect(scope.counter).toBe(1);
    scope.someValue = 'b';
    expect(scope.counter).toBe(1);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```
首先, 设置两个属性在 scope 上, 一个字符串, 另一个数字. 然后添加一个监听器去监控字符串, 并且当字符串改变时, 为数字加1. 我们期望第一次调用 $digest 的时候 counter 加1, 而后的每一次调用 $digest, 如果值改变, 则 counter 加1.

注意, 我们也指定了相应的 listener 函数, 和 watch 函数一样, 它也将 scope 作为自己的参数, 另外还有监听器中的新值和旧值, 这样做, 让开发者很容易检查到变化.

为了实现上面的功能, $digest 需要记住, 每个 watch 函数上次的返回值. 我们已经为每个监听器创建了对象, 可以将上次返回值存储在各自的监听器对象里. 下面是 '$digest' 的新定义:

```js
Scope.prototype.$digest = function() {
    var self = this;
    var newValue, oldValue;
    _.forEach(this.$$watchers, function(watcher) {
        newValue = watcher.watchFn(self);
        oldValue = watcher.last;
        if (newValue !== oldValue) {
            watcher.last = newValue;
            watcher.listenerFn(newValue, oldValue, self);
        }
}); };
```
对于每一个监听器, 我们比较 watch 函数的返回值和之前存储在 last 属性上的值. 如果两值不同, 我们调用 listener 函数, 并将新值, 旧值和当前的 scope 作为参数传入. 最后, 我们将新值设置给监听器对象的 last 属性, 用于下次比较.

现在, 我们已经实现了 AngularJS scope 的本质: 添加监控并并在 digest 循环中运行.

我们已经可以看到 AngularJS scope 中一些重要的性能特征:

- 在 Scope 上添加数据, 并不会对性能产生影响, 如果没有监听器去监控属性, 无论是否在scope上都无所谓. AngularJS 不会遍历 scope 的属性, 它遍历监听器.
- 在每个 `$degist` 中调用 watch 函数, 基于这个原因, 最好关注 watch 函数的数量, 还有每个单独 watch 或表达式的性能.