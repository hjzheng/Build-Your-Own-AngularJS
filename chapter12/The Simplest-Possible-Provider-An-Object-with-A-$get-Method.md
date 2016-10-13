### 一个尽可能最简单的 Provider: 拥有 $get 方法的对象

一般来说, Angular 调用的 provider 是一个拥有 $get 方法的任意 JavaScript 对象. 当你将这样一个对象传给 injector, injector 会调用 $get 方法并将它的返回值作为实际的依赖:

```js
it('allows registering a provider and uses its $get', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', {
        $get: function() {
            return 42;
        }
    });
    var injector = createInjector(['myModule']);
    expect(injector.has('a')).toBe(true);
    expect(injector.get('a')).toBe(42);
});
```

因为 a 的 provider 是对象 `{$get: function() { return 42 }}`, 所以, 上面测试中的 a 是 42.

我们现在正在使用的这种间接的方式不是非常有用, 但是正如我们将要看到的, 它给了我们一个方便配置 a 的机会, 这是 constant 不可能有的.

为了实现这种 provider, 让我们从模块加载器开始, 它需要一个方法去注册 provider, 这个方法必须把注册的调用信息放到  invoke queue 中:

```js
var moduleInstance = {
    name: name,
    requires: requires,
    constant: function(key, value) {
        invokeQueue.push(['constant', [key, value]]);
    },
    provider: function(key, provider) {
        invokeQueue.push(['provider', [key, provider]]);
    },
    _invokeQueue: invokeQueue
};
```

在组件注册中, 有一些重复代码结构, 所以让我们引入一个通用的 invokeLater 函数, 该函数可以以最小的代价实现 module.constant 和 module.provider.

```js
var createModule = function(name, requires, modules) {
    if (name === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid module name';
    }

    var invokeQueue = [];
    var invokeLater = function(method) {
        return function() {
            invokeQueue.push([method, arguments]);
            return moduleInstance;
        };
    };

    var moduleInstance = {
        name: name,
        requires: requires,
        constant: invokeLater('constant'),
        provider: invokeLater('provider'),
        _invokeQueue: invokeQueue
    };

    modules[name] = moduleInstance;

    return moduleInstance;
};
```

invokeLater 返回一个函数, 是一个已经被预先注册特定类型的应用组件, 更确切的说是一个特定的 $provide 方法的函数. 该函数会被 push 到 invoke queue.

注意, 在注册的时候, 我们也返回模块实例, 这样就使链式注册成为可能: module.constant(‘a’, 42’).constant(‘b’, 43).

在 injector 的 $provide 对象, 我们仍然需要代码去处理 invoke queue 中的 provider. 现在, 我们简单的调用 provider 的 $get 方法 和 将返回的值放到 cache 中, 让我们满足第一个测试用例:

```js
var $provide = {
    constant: function(key, value) {
        if (key === 'hasOwnProperty') {
            throw 'hasOwnProperty is not a valid constant name!';
        }
        cache[key] = value;
    },
    provider: function(key, provider) {
        cache[key] = provider.$get();
    }
};
```
