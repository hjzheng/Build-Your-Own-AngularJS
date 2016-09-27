### 阻止非必要的对象遍历

现在, 我们对 Object 进行了两次遍历, 对于一个非常巨大的 Object, 它的开销是非常昂贵的.
Since we’re working within a watch function that gets executed in every single digest, we need to take care not to do too much work.

基于这个原因, 我们对 Object 变化检测应用一个重要的优化.

首先, 我们将持续跟踪新旧对象的大小:

- 对于老对象, 我们保持一个变量, 当添加属性, 变量自增, 当删除属性, 变量自减.

- 对于新对象, 我们计算它的大小, 在第一个循环中.

当第一次循环结束时, 我们就知道当前两个对象的大小. 然后, 如果旧对象的大小大于新对象的, 继续第二个循环. 如果大小相等, 跳过第二个循环.

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var oldLength;
    var changeCount = 0;
    var internalWatchFn = function(scope) {
        var newLength;
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
                    oldLength = 0;
                }
                newLength = 0;
                _.forOwn(newValue, function(newVal, key) {
                    newLength++;
                    if (oldValue.hasOwnProperty(key)) {
                        var bothNaN = _.isNaN(newVal) && _.isNaN(oldValue[key]);
                        if (!bothNaN && oldValue[key] !== newVal) {
                            changeCount++;
                            oldValue[key] = newVal;
                        }
                    } else {
                        changeCount++;
                        oldLength++;
                        oldValue[key] = newVal;
                    }
                });
                if (oldLength > newLength) {
                    changeCount++;
                    _.forOwn(oldValue, function(oldVal, key) {
                        if (!newValue.hasOwnProperty(key)) {
                            oldLength--;
                            delete oldValue[key];
                        }
                    });
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

    var internalListenerFn = function() {
        listenerFn(newValue, oldValue, self);
    };

    return this.$watch(internalWatchFn, internalListenerFn);
};
```

注意: 我们处理添加新属性和改变属性是不同的, 因为添加新属性, 需要为 oldLength 变量进行自增.
