### 检测数组变化: 初始化数组时

$watchCollection 内部的 watch 函数将会有两个顶层条件分支: 一个处理对象, 两一个处理非对象. 因为 JavaScript 数组也是对象, 所以我们将在第一个分支中处理数组. 然而在该分支内需要一个嵌套分支, 来对数组和其他对象做不同的处理.

现在, 我们可以通过 LoDash 的 _.isObject 和 _.isArray 函数去判断监控的值是数组还是对象.

```js
var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
    if (_.isObject(newValue)) {
        if (_.isArray(newValue)) {

        } else {

        }
    } else {
        if (!self.$$areEqual(newValue, oldValue, false)) {
            changeCount++;
        }
        oldValue = newValue;
    }
    return changeCount;
};
```

第一件事, 我们可以检测 oldValue 是否是数组, 如果不是, 显然值发生了变化.

```js
it('notices when the value becomes an array', function() {
    scope.counter = 0;
    scope.$watchCollection(
        function(scope) { return scope.arr; },
        function(newValue, oldValue, scope) {
            scope.counter++;
        }
    );
    scope.$digest();
    expect(scope.counter).toBe(1);
    scope.arr = [1, 2, 3];
    scope.$digest();
    expect(scope.counter).toBe(2);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```

因为我们的数组条件分支是空的, 我们不会注意到这种改变.

```js
var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
    if (_.isObject(newValue)) {
        if (_.isArray(newValue)) {
            if (!_.isArray(oldValue)) {
                changeCount++;
                oldValue = [];
            }
        } else {

        }
    } else {
        if (!self.$$areEqual(newValue, oldValue, false)) {
            changeCount++;
        }
        oldValue = newValue;
    }
    return changeCount;
};
```

如果 oldValue 不是数组, 我们记录该变化. 同时将 oldValue 初始化成空数组.
