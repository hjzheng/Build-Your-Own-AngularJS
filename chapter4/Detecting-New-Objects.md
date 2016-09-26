### 检测对象变化: 初始化对象

让我们将注意力转向对象, 或者更精确些, 不是数组和类数组对象的对象, 例如下面的字典

```js
{
    aKey: 'aValue',
    anotherKey: 42
}
```

检测对象中变化的方式, 与我们在数组中所做的类似. 对象变化的检测的实现变得简单一点, 因为没有类对象对象使检测复杂化. 另一方面, 我们需要在对象变化检测上做的多一些, 因为我们不需要处理 length 属性.

首先, 像数组一样, 确保覆盖一个不是对象的值变成对象的情况:

```js
it('notices when the value becomes an object', function() {
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.obj = {a: 1};
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们使用数组中的做法处理这种情况的, 如果 oldValue 不是对象, 初始化一个, 并记录这次改变.

```js
var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
        if (_.isObject(newValue)) {
            if (isArrayLike(newValue)) {
                if (!_.isArray(oldValue)) {
                    changeCount++;
                    oldValue = [];
                }
                if (newValue.length !== oldValue.length) {
                    changeCount++;
                    oldValue.length = newValue.length;
                }
                _.forEach(newValue, function(newItem, i) {
                    var bothNaN = _.isNaN(newItem) && _.isNaN(oldValue[i]);
                    if (!bothNaN && newItem !== oldValue[i]) {
                        changeCount++;
                        oldValue[i] = newItem;
                    }
                });
            } else {
                if (!_.isObject(oldValue) || isArrayLike(oldValue)) {
                    changeCount++;
                    oldValue = {};
                }
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

注意, 数组也是对象, 我们不能只用 _.isObject 判断 oldValue. 我们也需要用 isArrayLike 排除数组和类数组对象.
