### 检测对象变化: 删除对象的属性

剩下的操作就是删除对象的属性:

```js
it('notices when an attribute is removed from an object', function() {
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
    delete scope.obj.a;
    scope.$digest();
    expect(scope.counter).toBe(2);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```

对于删除数组元素, 我们通过截断内部数组, 并且同时遍历两个数组, 检测数组中所有元素是否相同. 对于删除对象上属性却不能这样做. 检测一个对象上的属性是否被删除, 我们需要第二个循环. 这次我们遍历旧对象上的属性, 看看属性是否还在新对象上.
如果不在, 说明该属性已经不存在了, 并且我们还要从内部对象上删除该属性.

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
            _.forOwn(oldValue, function(oldVal, key) {
                if (!newValue.hasOwnProperty(key)) {
                    changeCount++;
                    delete oldValue[key];
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
