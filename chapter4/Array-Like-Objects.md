### 类数组对象

我们已经处理了数组, 但是还有一种特殊的情况需要考虑:

除了真正的数组(继承自 Array.prototype)之外, JavaScript 环境中有一些行为类似于数组, 但实际上又不是数组的对象. Angular 的 $watchCollection 把这类对象当作数组对待, 因此我们也这样做.

一种类数组对象就是每个函数都有的本地变量 arguments, 它包含了函数被调用时的参数. 让我们通过一个测试用例来检查是否支持这种类数组对象:

```js
it('notices an item replaced in an arguments object', function() {
    (function() {
        scope.arrayLike = arguments;
    })(1, 2, 3);
    scope.counter = 0;
    scope.$watchCollection(
        function(scope) { return scope.arrayLike; },
        function(newValue, oldValue, scope) {
            scope.counter++;
        }
    );
    scope.$digest();
    expect(scope.counter).toBe(1);
    scope.arrayLike[1] = 42;
    scope.$digest();
    expect(scope.counter).toBe(2);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```

我们用一些参数去调用一个匿名函数, 将匿名函数的 arguments 存储在 Scope 上. 我们检查 arguments 对象的变化是否被 $watchCollection 注意到.

另一种类数组对象就是 DOM 的 NodeList, 你可以通过一些 DOM 操作得到它, 例如 querySelectorAll 和 getElementsByTagName. 让我们也来测试它吧.

```js
it('notices an item replaced in a NodeList object', function() {
    document.documentElement.appendChild(document.createElement('div'));
    scope.arrayLike = document.getElementsByTagName('div');
    scope.counter = 0;
    scope.$watchCollection(
        function(scope) { return scope.arrayLike; },
        function(newValue, oldValue, scope) {
            scope.counter++;
        }
    );
    scope.$digest();
    expect(scope.counter).toBe(1);
    document.documentElement.appendChild(document.createElement('div'));
    scope.$digest();
    expect(scope.counter).toBe(2);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```

首先, 我们在 DOM 上添加一个 div 元素, 然后调用 document 上的 getElementsByTagName 拿到一个 NodeList 对象, 在 Scope 上存放该 NodeList 对象并添加一个监听器监控它.
当我们想要引起该 NodeList 变化, 只需要在 DOM 上添加两一个 div 元素. 该 NodeList 会立即增加一个新 div 元素. 它被称为 'live collection'.

我们检查 $watchCollection 是否可以检测到这种变化.

结果, 两个测试都失败了, 问题是 LoDash 的 _.isArray 方法, 只能检测数组, 而不能检测类数组. 因此我们需要创建适合测试用例的断定函数, 做类型检测.

```js
function isArrayLike(obj) {
    if (_.isNull(obj) || _.isUndefned(obj)) {
        return false;
    }
    var length = obj.length;
    return _.isNumber(length);
}
```

该函数用一个对象作参数, 返回一个布尔值, 用来断定传入的对象参数是否是类数组对象. 我们通过确定该对象存在, 并且有一个数字型的 length 属性来确定对象是否是类数组.

现在, 我们所需要做的是在 $watchCollection 函数调用新的断定函数而不是 _.isArray.
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

注意, 当我们处理这种类数组对象时, 内部的 oldValue 永远是一个数组, 不可能是类数组对象.
