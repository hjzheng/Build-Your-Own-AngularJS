### $apply - 使用脏检查来整合外部代码引起的数据变化
大概，Scope 所拥有的方法中，最为著名的就是 $apply 了，它被认为是向 angular 中整合由外部行为导致的数据变化的标准方法。

关于这样做的原因，下面的内容将是一个很好的解释：

$apply 方法接收一个函数作为参数，首先使用 $eval 来调用这个函数，接着通过调用 $digest 来发起一轮脏检查循环，下面是一个针对这个过程的测试用例：

```js
// test / scope_spec.js
describe('$apply', function () {
    var scope;
    beforeEach(function () {
        scope = new Scope();
    });
    it('executes the given function and starts the digest', function () {
        scope.aValue = 'someValue';
        scope.counter = 0;
        scope.$watch(
            function (scope) {
                return scope.aValue;
            },
            function (newValue, oldValue, scope) {
                scope.counter++;
            }
        );
        scope.$digest();
        expect(scope.counter).toBe(1);
        scope.$apply(function (scope) {
            scope.aValue = 'someOtherValue';
        });
        expect(scope.counter).toBe(2);
    });
});
```
上面的用例中，我们针对 scope.aValue 设立了一个 watch(监听)，并且在对应的 listener(监听器) 中通过 scope.counter 进行了计数。我们期望当 $apply 方法被调用时，这个监听能立刻被触发。

为了使我们的测试用例通过，我们可以对 $apply 做一个简单的实现：

```js
// src/scope.js
Scope.prototype.$apply = function(expr) {
    try {
        return this.$eval(expr);
    } finally {
        this.$digest();
    }
};
```
在 finally 区域中我们调用了 $digest 方法，这样做可以确保，即使当你所执行的 expr 发生错误并抛出异常时，脏检查依然会被发起。

当我们在 angular 没有掌控的范围内运行了一段代码，这些代码恰恰可能会引起 Scope 的变化，这时，只要我们使用 $apply 来“包裹”着运行该段代码，就可以保证所有发生了变化的数据所对应的监听被触发。
通常，人们口中所说的，使用 $apply 将代码的运行整合到“Angular 的生命周期中”，本质上来说正是这样的逻辑，并没有比这深奥很多。
