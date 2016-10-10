### 为注入的函数绑定 this 对象

有时, 我们想要注入的函数实际上是对象上的方法. 在这类方法中, this 的值通常是需要注意的. 当你直接调用时, JavaScript 语言小心处理绑定的 this, 但是通过 injector.invoke 调用, this 并没有自动绑定

然而, 我们可以给 injector.invoke 方法一个 this 值作为第二个可选参数, 当调用方法时, 这个 this 值会被绑定.

```js
it('invokes a function with the given this context', function() {
    var module = window.angular.module('myModule', []);

    module.constant('a', 1);

    var injector = createInjector(['myModule']);

    var obj = {
        two: 2,
        fn: function(one) { return one + this.two; }
    };

    obj.fn.$inject = ['a'];

    expect(injector.invoke(obj.fn, obj)).toBe(3);
});
```

因为已经使用 Function.apply 调用函数了, 所以只需要将 this 值传给 Function.apply:

```js
function invoke(fn, self) {
    var args = _.map(fn.$inject, function(token) {
        if (_.isString(token)) {
            return cache[token];
        } else {
            throw 'Incorrect injection token! Expected a string, got '+token;
        }
    });

    return fn.apply(self, args);
}
```
