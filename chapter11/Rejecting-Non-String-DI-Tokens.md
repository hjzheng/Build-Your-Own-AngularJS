### 拒绝非字符串类型的 DI 依赖

我们已经看到, $inject 数组应该包含依赖名称. 如果在 $inject 数组中放入无效东西, 例如 放入一个数字, 当前的实现仅会匹配一个 undefined. 此时我们需要抛出一个异常, 让使用者知道他们做错了:

```js
it('does not accept non-strings as injection tokens', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);

    var injector = createInjector(['myModule']);

    var fn = function(one, two) { return one + two; };
    fn.$inject = ['a', 2];

    expect(function() {
        injector.invoke(fn);
    }).toThrow();
});
```

在遍历依赖的 mapping 函数中, 通过一个简单的类型检查就可以搞定:

```js
function invoke(fn) {
    var args = _.map(fn.$inject, function(token) {
        if (_.isString(token)) {
            return cache[token];
        } else {
            throw 'Incorrect injection token! Expected a string, got '+token;
        }
    });

    return fn.apply(null, args);
}
```
