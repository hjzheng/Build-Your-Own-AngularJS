### 对 listener 函数处理旧集合的值

listener 函数的定义, 它会获取三个参数: watch 函数返回的 newValue, watch 函数上次返回的 oldValue, 当前 Scope.
本章, 我们已经按照定义提供这些值, 但是提供的值是有问题的, 特别是牵扯到 oldValue 时.

问题是在 innernalWatchFn 中维持的旧值, 在调用 listener 函数时已经被更新成新值. 传给 listener 函数的新旧值是一样的.

因此下面对非集合的测试用例是无法通过的:

```js
it('gives the old non-collection value to listeners', function() {
  scope.aValue = 42;
  var oldValueGiven;
  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );
  scope.$digest();
  scope.aValue = 43;
  scope.$digest();
  expect(oldValueGiven).toBe(42);
});
```

这是一个对 Array 的测试用例:

```js
it('gives the old array value to listeners', function() {
  scope.aValue = [1, 2, 3];
  var oldValueGiven;
  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      oldValueGiven = oldValue;
    }
  );
  scope.$digest();
  scope.aValue.push(4);
  scope.$digest();
  expect(oldValueGiven).toEqual([1, 2, 3]);
});
```

还有一个对 Object 的测试用例:

```js
it('gives the old object value to listeners', function() {
    scope.aValue = {a: 1, b: 2};
    var oldValueGiven;
    scope.$watchCollection(
      function(scope) { return scope.aValue; },
      function(newValue, oldValue, scope) {
        oldValueGiven = oldValue;
      }
    );
    scope.$digest();
    scope.aValue.c = 3;
    scope.$digest();
    expect(oldValueGiven).toEqual({a: 1, b: 2});
});
```

目前值的比较和复制的实现工作不错, 因此我们不想改变它. 我们引入一个变量, 叫它 veryOldValue, 它会保持一个旧集合的 copy, 并且不会 innernalWatchFn 改变它.

维护 veryOldValue 需要 copy 数组或对象, 这是非常昂贵的. 我们不用每次都 copy 整个集合. 只在我们需要的时候, 进行 copy.

我们检查看看用户传给我们的 listener 函数是否需要至少两个参数:

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var oldLength;
    var veryOldValue;
    var trackVeryOldValue = (listenerFn.length > 1);
    var changeCount = 0;
    // ...
};
```

函数的 length 属性包含函数声明的参数个数, 如果它大于 1, 也就是 (newValue, oldValue), 或 (newValue, oldValue, scope), 只有需要的时候, 我们才去追踪
veryOldValue.

注意, 这不会引起复制 veryOldValue 的代价, 除非你在 listener 函数中声明 oldValue 参数, 也不会映射到 listener 函数的 arguments 对象中. 实际上, 你需要声明它

剩下的工作, 发生在 internalListenerFn, 用 veryOldValue 替换 oldValue, 接着需要 copy 当前值作为下一次的 veryOldValue.

```js
var internalListenerFn = function() {
    listenerFn(newValue, veryOldValue, self);
    if (trackVeryOldValue) {
        veryOldValue = _.clone(newValue);
    }
};
```

在章节 1 我们讨论了 oldValue 在 listener 函数第一次调用时值, 对这次调用, 它必须被定义成 newValue.

```js
it('uses the new value as the old value on frst digest', function() {
scope.aValue = {a: 1, b: 2};
var oldValueGiven;
scope.$watchCollection(
function(scope) { return scope.aValue; },
function(newValue, oldValue, scope) {
oldValueGiven = oldValue;
}
);
scope.$digest();
expect(oldValueGiven).toEqual({a: 1, b: 2});
});
```

这个测试不会通过, 因为 oldValue 是 undefined 的, 因为在第一次调用前, 我们还没有给 veryOldValue 赋值, 需要设置一个标志标示我们是否在第一次调用中, 基于它对 listener 进行不同的调用:

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var oldLength;
    var veryOldValue;
    var trackVeryOldValue = (listenerFn.length > 1);
    var changeCount = 0;
    var frstRun = true;
    // ...
    var internalListenerFn = function() {
        if (frstRun) {
            listenerFn(newValue, newValue, self);
            frstRun = false;
        } else {
            listenerFn(newValue, veryOldValue, self);
        }
        if (trackVeryOldValue) {
            veryOldValue = _.clone(newValue);
        }
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```
