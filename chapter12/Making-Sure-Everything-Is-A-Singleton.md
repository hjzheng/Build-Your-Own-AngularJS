### 确保一切都是单例

你或许听过 "Angular 中的一切都是单例的", 这是真的. 当你在不同的地方使用相同的依赖, 你将会有一个引用指向相同的对象.

当前 injector 实现不是这样工作的. 当你请求创建两次 provider 时, 你会得到两个结果.

```js
it('instantiates a dependency only once', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', {$get: function() { return {}; }});

    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(injector.get('a'));
});
```

测试失败, 是因为, 我们调用了两次 $get, 每一次都会它都返回一个新对象给我们.

解决方法很简单, 只需将 provider 调用后的返回值放到 instanceCache 中. 下次, 需要相同的依赖时, 可以直接在 instanceCache 中拿到, 我们永远不会调用同一个 provider 的 $get 方法两次:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        var provider = providerCache[name + 'Provider'];
        var instance = instanceCache[name] = invoke(provider.$get);
        return instance;
    }
}
```
