### 检测数组变化: 替换或重排元素时

这还有一类我们必须检测的变化: 当数组中的元素发生替换或重新排序且数组的长度并未发生时.

```js
it('notices an item replaced in an array', function() {
  scope.arr = [1, 2, 3];
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.arr[1] = 42;
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});

it('notices items reordered in an array', function() {
  scope.arr = [2, 1, 3];
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.arr.sort();
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

检测像这样的变化, 需要遍历整个数组并在每一个索引上比较新旧数组的元素. 这样做会关注到数组中元素替换和重新排序的值. 当遍历时, 我们还需要同步旧值和新值的内容.

```js
var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
    if (_.isObject(newValue)) {
        if (_.isArray(newValue)) {
            if (!_.isArray(oldValue)) {
                changeCount++;
                oldValue = [];
            }
            if (newValue.length !== oldValue.length) {
                changeCount++;
                oldValue.length = newValue.length;
            }
            _.forEach(newValue, function(newItem, i) {
                if (newItem !== oldValue[i]) {
                    changeCount++;
                    oldValue[i] = newItem;
                }
            });
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

我们使用 LoDash 的 _.forEach 方法遍历新数组. 它为我们提供了每次循的元素和索引. 我们使用索引从旧数组获取相应的元素.

在章节 1 中, 我们看到 NaN 是如何成为问题, 因为 NaN 不等于 NaN. 我们已经普通的 watch 中特殊处理了. 同样也得在 watchCollection 中处理:

```js
it('does not fail on NaNs in arrays', function() {
  scope.arr = [2, NaN, 3];
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.arr; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

这个测试会抛出一个异常, 因为 NaN 永远都会触发一个改变, 因此无限 digest 循环. 我们来修复它吧!

```js
var internalWatchFn = function(scope) {
    newValue = watchFn(scope);
    if (_.isObject(newValue)) {
        if (_.isArray(newValue)) {
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

当前的实现, 我们可以检测到可能发生在数组上的任何变化, 并且无需复制, 甚至遍历数组内任何嵌套的数据结构.