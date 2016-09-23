### 检测数组变化: 添加和删除数组元素时

接下来, 当添加或删除数组元素时, 数组的长度会发生变化. 让我们来为每种操作添加些测试用例:

```js
it('notices an item added to an array', function() {
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
  scope.arr.push(4);
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
it('notices an item removed from an array', function() {
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
  scope.arr.shift();
  scope.$digest();
  expect(scope.counter).toBe(2);
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

在这两个测试用例中, 我们都监听 Scope 上的数组并且对数组进行了操作, 然后检测这种变化是否在 digest 时被注意到.

这种变化, 通过简单的比较新旧数组的长度就可以被检测到. 我们必须将新长度同步到内部 oldValue 数组上, 这里只是简单的做了个赋值.

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
这种将新长度赋值显然是不对的, 只是暂时通过当前测试, 如何解决, 请关注下一节