### 为注入的函数提供本地值

大部分情况, 你只想让 injector 为函数提供所有参数, 但是有种情况, 你想在函数调用时, 显示地提供一些参数. 这可能是因为你想覆盖一些参数, 或者使用一些没有被 injector 注册的参数.

基于这个目的, injector.invoke 需要第三个可选参数, 这个可选参数是一个映射关系(依赖名称到值)对象. 如果提供该参数, 优先从这个参数对象中进行依赖查找, 其次才是 injector 自身.

```js
it('overrides dependencies with locals when invoking', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);

    var injector = createInjector(['myModule']);

    var fn = function(one, two) { return one + two; };

    fn.$inject = ['a', 'b'];

    expect(injector.invoke(fn, undefned, {b: 3})).toBe(4);
});
```

在依赖映射函数中, 我们首先查找本地依赖, 如果没有找到, 在查找 cache 中的:

```js
function invoke(fn, self, locals) {
    var args = _.map(fn.$inject, function(token) {
        if (_.isString(token)) {
            return locals && locals.hasOwnProperty(token) ? locals[token] : cache[token];
        } else {
            throw 'Incorrect injection token! Expected a string, got '+token;
        }
    });

    return fn.apply(self, args);
}
```
