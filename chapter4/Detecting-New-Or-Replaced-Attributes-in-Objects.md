### 检测对象变化: 添加和替换对象属性

我们给对象添加一个新属性去触发一个变化.

```js
it('notices when an attribute is added to an object', function() {
  scope.counter = 0;
  scope.obj = {a: 1};
  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.obj.b = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

我们也可以触发一个变化, 当已存在的属性的值发生改变:

```js
it('notices when an attribute is changed in an object', function() {
  scope.counter = 0;
  scope.obj = {a: 1};
  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.obj.a = 2;
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

上面两种情况可以被用相同的方式处理, 遍历新对象上的所有属性, 检查是否有相同的值在旧对象上.

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
            _.forOwn(newValue, function(newVal, key) {
                if (oldValue[key] !== newVal) {
                    changeCount++;
                    oldValue[key] = newVal;
                }
            });
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

当遍历对象的时候, 我们将新对象的所有属性同步到旧对象上, 以便在下次 digest 中使用它们.

LoDash 的 _.forOwn 方法遍历对象的属性, 但是它只遍历对象自己定义的属性. 从原型链上继承的属性会被排除在外. 因此 $watchCollection 不监控对象上继承自原型链的属性.

和数组一样, NaN 需要被特殊对待. 如果一个对象有一个值是 NaN 的属性, 就会因此无限 digest:

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
            _.forOwn(newValue, function(newVal, key) {
                var bothNaN = _.isNaN(newVal) && _.isNaN(oldValue[key]);
                if (!bothNaN && oldValue[key] !== newVal) {
                    changeCount++;
                    oldValue[key] = newVal;
                }
            });
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
