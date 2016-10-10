### 依赖注入

现在, 我们有一个部分有用的应用组件 injector, 在其中, 我们可以加载东西, 也可以查找东西. 但是 injector 真正的目的实际上是去做依赖注入.
所谓依赖注入就是调用函数和构造器对象时, 自动查找它们需要的依赖. 本章剩余的部分, 我们将关注 injector 的依赖注入特性.

这是一个基本的想法: 我们给 injector 一个函数, 要求它去调用这个函数. 也期望 inject 找出函数需要什么参数并将这些参数提供给函数.
那么, injector 如何找出给定的函数需要什么参数呢? 最简单的方法是显示地提供参数信息, 使用一个叫做 $inject 属性. $inject 属性可以保存函数依赖名称的数组.

injector 会查找这些依赖, 并在调用函数时使用它们:

```js
it('invokes an annotated function with dependency injection', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);

    var injector = createInjector(['myModule']);

    var fn = function(one, two) { return one + two; };
    fn.$inject = ['a', 'b'];

    expect(injector.invoke(fn)).toBe(3);
});
```

通过简单从 injector cache 中查找 $inject 数组中的每一项去实现. injector cache 中存储这些依赖名称对应的值:

```js
function createInjector(modulesToLoad) {
    var cache = {};
    var loadedModules = {};

    var $provide = {
        constant: function(key, value) {
            if (key === 'hasOwnProperty') {
                throw 'hasOwnProperty is not a valid constant name!';
            }

            cache[key] = value;
        }
    };

    function invoke(fn) {
        var args = _.map(fn.$inject, function(token) {
            return cache[token];
        });
        return fn.apply(null, args);
    }

    _.forEach(modulesToLoad, function loadModule(moduleName) {
        if (!loadedModules.hasOwnProperty(moduleName)) {
            loadedModules[moduleName] = true;
            var module = window.angular.module(moduleName);
            _.forEach(module.requires, loadModule);
            _.forEach(module._invokeQueue, function(invokeArgs) {
                var method = invokeArgs[0];
                var args = invokeArgs[1];
                $provide[method].apply($provide, args);
            });
        }
    });

    return {
        has: function(key) {
            return cache.hasOwnProperty(key);
        },
        get: function(key) {
            return cache[key];
        },
        invoke: invoke
    };
}
```

这是你在使用 angular 时最低级别的依赖注明方式.
