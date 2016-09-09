### 在 Digest 时获得通知

在 Digest 时, 如果你想获得通知, 你只需要注册一个 watch, 而不需要 listener 函数, 因为每次 digest, 每个 watch 函数都会被执行. 让我们添加一个测试用例:

```js
it('may have watchers that omit the listener function', function() {
  var watchFn = jasmine.createSpy().and.returnValue('something');
  scope.$watch(watchFn);
  scope.$digest();
  expect(watchFn).toHaveBeenCalled();
});
```
watch 函数没有必要返回任何值, 但是可以返回, 就像上面测试用例一样. 当 scope digest 时, 我们当前的实现会抛出一个异常, 那是因为我们调用了一个不存在的函数. 为了支持这种用例, 我们需要检查 listener 函数是否省略, 如果省略, 放置一个空函数.

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn || function() { },
    last: initWatchVal
  };
  this.$$watchers.push(watcher);
};
```
如果使用这种方式, 请记住, 即使没有 listenerFn, AngularJS 也会查看 watchFn 的返回值. 如果返回一个值, 这个值会被做脏检查, 确保使用这种方式不会引起额外的工作, 如果没有返回值, 被监控的值都会变成 undefined .
