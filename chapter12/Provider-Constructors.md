### Provider 的构造函数

早先, 我们定义的 provider 是一个带有 $get 方法的对象. 这是正确的, 但是当你注册一个 provider, 你也可以选择使用构造函数实例化一个 provider:

```js
function AProvider() {
    this.$get = function() { return 42; }
}
```

这是一个构造函数, 当你实例化它时, 结果是一个带有 $get 方法的对象(provider), 另一种构造函数如下所示:

```js
function AProvider() {
    this.value = 42;
}

AProvider.prototype.$get = function() {
    return this.value;
}
```

所以, 使用这种构造函数风格, Angular 不会真正关系 $get 方法来自于哪里, 只要返回结果的对象有就行. 你可以充分利用 JavaScript 类式(使用 prototype 模拟)编程(或 ES2015 中的类)以及继承:

```js
it('instantiates a provider if given as a constructor function', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', function AProvider() {
        this.$get = function() { return 42; };
    });

    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(42);
});
```

更重要的是, 这些构造函数也可以被注入其他依赖:

```js
it('injects the given provider constructor function', function() {
    var module = window.angular.module('myModule', []);
    module.constant('b', 2);
    module.provider('a', function AProvider(b) {
        this.$get = function() { return 1 + b; };
    });

    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(3);
});
```

要启用构造函数的风格的 provider, 我们需要在注册时, 检查 provider 是否是一个函数. 如果是, 需要实例化它. 在上一章节, 我们创建一个函数 instantiate 是来做这个的.

```js
provider: function(key, provider) {
    if (_.isFunction(provider)) {
        provider = instantiate(provider);
    }
    providerCache[key + 'Provider'] = provider;
}
```
