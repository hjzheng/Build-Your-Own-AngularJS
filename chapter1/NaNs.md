### NaNs

在 JavaScript 中, NaN 是不等于它自己的. 这听起来有些奇怪, 这也正是原因所在. 如果我们不在脏检查函数中明确处理 NaN, 那么将 NaN 作为返回值的 watch 函数将永远是脏的.

基于值得脏检查已经通过 LoDash isEqual 方式被处理, 对于基于引用的检查也被我们自己搞定, 就差 NaN 的情况了:

```js
it('correctly handles NaNs', function() {
  scope.number = 0/0; // NaN
  scope.counter = 0;
  scope.$watch(
    function(scope) { return scope.number; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

我们监控一个 NaN 的值, 当它改变的时候, 对 counter 加 1. 我们期望在第一次 $digest 时, counter 加 1, 随后的 $digest, counter 保持不变. 然而, 运行该测试, 我们却得到了 TTL reached 的异常.
这个 scope 永远都不会稳定, 因为 NaN 永远不可能等于 NaN.

让我们对 $$areEqual 稍作调整:

```js
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  if (valueEq) {
    return _.isEqual(newValue, oldValue);
  } else {
    return newValue === oldValue ||
        (typeof newValue === 'number' && typeof oldValue === 'number' &&
        isNaN(newValue) && isNaN(oldValue));
  }
};
```