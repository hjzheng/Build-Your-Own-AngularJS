### 严格模式

从源码中确认函数依赖方式是 Angular 最具争议的特性之一. 在一定程度上是因为它的不可以预期和神奇性. 但是使用这种方式仍然有一些非常实际的问题: 在发布前, 如果你选择压缩你的 JavaScript 代码: 大部分人都会这样做, 应用源码会被改变, 关键是当你使用例如 UglifyJS 或 Closure Compiler 这样的压缩工具时, 函数的参数名称也会改变. 这样会破坏掉未标注的依赖注入.

因为这个原因, 许多人不会选择使用未标注依赖注入方式的特性. Angular 可以通过强制执行严格的依赖注入模式来帮助坚持这个决定, 如果你尝试注入一个没有明确标注依赖的函数，它会抛出一个错误。

通过将布尔标志作为第二个参数传给 createInjector 函数启用严格的依赖注入:

```js
it('throws when using a non-annotated fn in strict mode', function() {
    var injector = createInjector([], true);
    var fn = function(a, b, c) { };
    expect(function() {
        injector.annotate(fn);
    }).toThrow();
});
```

createInjector 函数接受第二个参数, 只有为 true 时, 启用严格模式:

```js
function createInjector(modulesToLoad, strictDi) {
    var cache = {};
    var loadedModules = {};
    strictDi = (strictDi === true);
    // ...
}
```

在 annotate 方法中, 当我们获得解析到的函数参数时, 我们要检查如果是严格模式并抛出异常:

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
        return fn.$inject;
    } else if (!fn.length) {
        return [];
    } else {
        if (strictDi) {
            throw 'fn is not using explicit annotation and '+
                'cannot be invoked in strict mode';
        }
        var source = fn.toString().replace(STRIP_COMMENTS, '');
        var argDeclaration = source.match(FN_ARGS);
        return _.map(argDeclaration[1].split(','), function(argName) {
            return argName.match(FN_ARG)[2];
        });
    }
}
```
