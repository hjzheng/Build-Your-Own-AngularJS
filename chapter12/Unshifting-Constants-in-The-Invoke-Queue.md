### 将 constant 放在 invoke queue 最前面

正如之前所看到的, 实例构建的延迟是一个很好的特性, 这样应用开发者不必再按照它们依赖的顺序去注册应用组件. 你可以在 B 之后注册 A, 即使 A 依赖 B.

对于 provider 构造函数, 就没有那么自由了. 当 provider 被注册时, provider 的构造函数被调用, 如果 BProvider 依赖 AProvider, 那么 A 需要在 B 之前注册.

对于 constant, Angular 确实帮了你一些. constant 将永远被注册在最前面, 所以你可以有一个依赖之后注册的 constant 的 provider.

```js
it('registers constants frst to make them available to providers', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', function AProvider(b) {
        this.$get = function() { return b; };
    });

    module.constant('b', 42);
    var injector = createInjector(['myModule']);
        expect(injector.get('a')).toBe(42);
    });
```

当 constant 被注册在一个模块上时, 模块加载器总是将它们添加 invoke queue 的前面. 这是它们为何被最先注册的原因. 这是 Angular 对队列的安全排序, 因为 constant 不会依赖任何其它东西.

如何工作呢? 模块加载器的 invokeLater 函数需要一个可选参数去制定数组使用哪种方法添加队列元素. 默认是 push, 它会添加元素到队列的最后面:

```js
var invokeLater = function(method, arrayMethod) {
    return function() {
        invokeQueue[arrayMethod || 'push']([method, arguments]);
        return moduleInstance;
    };
};
```

constant 的注册会使用 unshift 覆盖默认的 push 方法, 但是 provider 的注册使用默认的 push 方法.

```js
var moduleInstance = {
    name: name,
    requires: requires,
    constant: invokeLater('constant', 'unshift'),
    provider: invokeLater('provider'),
    _invokeQueue: invokeQueue
};
```
