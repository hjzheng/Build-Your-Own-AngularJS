### 对 $get 方法注入依赖

在这点上, 我们通过介绍 provider 获得的很少, 我们仅仅隐藏 $get 方法之后的实际依赖.

当我们想到一种构建的应用组件有自己的依赖的情况, 也就是说当我们的依赖有依赖的时候, 好处开始显现出来了. 目前, 我们没有办法处理这种情况, 但是我们有 $get 方法, 我们可以用依赖注入的方式调用它:

```js
it('injects the $get method of a provider', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.provider('b', {
        $get: function(a) {
            return a + 2;
        }
    });

    var injector = createInjector(['myModule']);
    expect(injector.get('b')).toBe(3);
});
```

这个很容易: 我们需要用依赖注入的方式调用 provider.$get 在上一章节, 我们已经准确实现该功能: invoke

```js
var $provide = {
    constant: function(key, value) {
        if (key === 'hasOwnProperty') {
            throw 'hasOwnProperty is not a valid constant name!';
        }
        cache[key] = value;
    },

    provider: function(key, provider) {
        cache[key] = invoke(provider.$get, provider);
    }
};
```

注意, 我们在调用中将 this 绑定到 provider 对象上. 因为 $get 是 provider 对象的一个方法, 它的 this 应该绑到 provider 对象上.

所以, invoke 不仅是一个可以调用你自己的函数的 injector 的方法, 而且也可以内部使用, 对 provider.$get 方法注入依赖.
