### Two Injectors: The Provider Injector and The Instance Injector

两个 injector 第一个区别是, 你可以对 provider 的构造函数注入其他依赖:

```js
it('injects another provider to a provider constructor function', function() {
var module = window.angular.module('myModule', []);
    module.provider('a', function AProvider() {
        var value = 1;
        this.setValue = function(v) { value = v; };
        this.$get = function() { return value; };
    });

    module.provider('b', function BProvider(aProvider) {
        aProvider.setValue(2);
        this.$get = function() { };
    });

    var injector = createInjector(['myModule']);
    expect(injector.get('a')).toBe(2);
});
```

目前, 我们只能注入实例, 例如 constant 和 provider 包含 $get 方法的返回值. 这里, 我们注入了一个 provider: provider b 有一个依赖 provider a, 我们期望 aProvider 被注入. 然后调用它的 setValue() 方法去配置 aProvider. 当我们通过 injector 获得实例时, 我们看到的方法调用实际已经发生了.

为了使我们测试通过, 在 getService 中添加一个查找 providerCache

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        if (instanceCache[name] === INSTANTIATING) {
            throw new Error('Circular dependency found: ' + name + ' <- ' + path.join(' <- '));
        }
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name)) {
        return providerCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        path.unshift(name);
        instanceCache[name] = INSTANTIATING;
        try {
            var provider = providerCache[name + 'Provider'];
            var instance = instanceCache[name] = invoke(provider.$get);
            return instance;
        } finally {
            path.shift();
            if (instanceCache[name] === INSTANTIATING) {
                delete instanceCache[name];
            }
        }
    }
}
```

我们现在为了两种不同的目的去查找 providerCache: 第一种, 我们查找 provider 为了实例化一个实例, 第二种仅仅查找一个 provider.
不管怎样, 它没有那么简单. 结果是, 你不能在任何你想注入的地方注入 provider 或 instance. 例如, 当你对一个 provider 的构造函数注入一个 provider 时, 你就不能注入一个实例在哪儿:

```js
it('does not inject an instance to a provider constructor function', function() {
    var module = window.angular.module('myModule', []);
        module.provider('a', function AProvider() {
        this.$get = function() { return 1; };
    });

    module.provider('b', function BProvider(a) {
        this.$get = function() { return a; };
    });

    expect(function() {
        createInjector(['myModule']);
    }).toThrow();
});
```

所以, 虽然 BProvider 依赖于 aProvider, 但是它不依赖于 a. 同样的, 虽然你可以对 $get 方法注入一个实例, 但是你不能在这里对它注入 provider:

```js
it('does not inject a provider to a $get function', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', function AProvider() {
        this.$get = function() { return 1; };
    });

    module.provider('b', function BProvider() {
        this.$get = function(aProvider) { return aProvider.$get(); };
    });

    var injector = createInjector(['myModule']);
    expect(function() {
        injector.get('b');
    }).toThrow();
});
```

你也不能使用 injector.invoke 调用一个被注入 provider 的函数:

```js
it('does not inject a provider to invoke', function() {
    var module = window.angular.module('myModule', []);
        module.provider('a', function AProvider() {
        this.$get = function() { return 1; }
    });

    var injector = createInjector(['myModule']);

    expect(function() {
        injector.invoke(function(aProvider) { });
    }).toThrow();
});
```

也不能通过调用 injector.get 获取并访问一个 provider:

```js
it('does not give access to providers through get', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', function AProvider() {
        this.$get = function() { return 1; };
    });

    var injector = createInjector(['myModule']);

    expect(function() {
        injector.get('aProvider');
    }).toThrow();
});
```
我们用这些测试概括出两种类型的注入之间的一个清晰的分界: 一种注入只会发生 provider 的构造函数之间并且只能注入其他 provider. 另一种发生在 $get 方法之间并且外部的 injector API 只能处理实例. 实例或许是被 provider 创建的, 但是 provider 不会被暴露出来.

这种分界实际上是通过两个单独的 injector 实现: 一个专门处理 provider, 另一个专门处理 instance. 后者将被通过公共 API 暴露出去, 前者只能被在 createInjector 内部使用.

让我们重新组织代码去支持两种 injector. 首先, 我们将一步一步介绍代码的变化, 最后列出所有重组后的 createInjector 的源码.

首先, 我们将会在 createInjector 里有一个内部函数用于处理两种内部的 injector. 这个函数有两个参数: 一个是用于查找依赖的 cache, 另一个当 cache 中没有依赖时, 用于回退的工厂方法:

```js
function createInternalInjector(cache, factoryFn) {

}
```

我们需要将所有的依赖查找函数移动到 createInternalInjector 中, 因为它们需要 cache 和 factoryFn 在作用域中. 首先, getService 已经在 createInternalInjector 中了, 它可以从给定 cache 中查找依赖:

```js
function createInternalInjector(cache, factoryFn) {
    function getService(name) {
        if (cache.hasOwnProperty(name)) {
            if (cache[name] === INSTANTIATING) {
                throw new Error('Circular dependency found: ' +
                name + ' <- ' + path.join(' <- '));
            }
            return cache[name];
        } else {
            path.unshift(name);
            cache[name] = INSTANTIATING;
            try {
                return (cache[name] = factoryFn(name));
            } finally {
                path.shift();
                if (cache[name] === INSTANTIATING) {
                    delete cache[name];
                }
            }
        }
    }
}
```

注意, 我们不再需要在 else 分支显示调用 provider. 这个工作发生在, 当缓存中没有时, 委托给 factoryFn 赋给 createInternalInjector.

invoke 和 instantiate 方法也需要移动到 createInternalInjector 中, 因为它们依赖 getService. 它们自己的实现不需要改变.

```js
function createInternalInjector(cache, factoryFn) {
    function getService(name) {
        if (cache.hasOwnProperty(name)) {
            if (cache[name] === INSTANTIATING) {
                throw new Error('Circular dependency found: ' +
                    name + ' <- ' + path.join(' <- '));
            }
            return cache[name];
        } else {
            path.unshift(name);
            cache[name] = INSTANTIATING;
            try {
                return (cache[name] = factoryFn(name));
            } finally {
                path.shift();
                if (cache[name] === INSTANTIATING) {
                    delete cache[name];
                }
            }
        }
    }

    function invoke(fn, self, locals) {
        var args = annotate(fn).map(function(token) {
        if (_.isString(token)) {
            return locals && locals.hasOwnProperty(token) ?
                locals[token] :
                getService(token);
            } else {
                throw 'Incorrect injection token! Expected a string, got '+token;
            }
        });

        if (_.isArray(fn)) {
            fn = _.last(fn);
        }
        return fn.apply(self, args);
    }

    function instantiate(Type, locals) {
        var instance = Object.create((_.isArray(Type) ? _.last(Type) : Type).prototype);
        invoke(Type, instance, locals);
        return instance;
    }
}
```

createInternalInjector 的最后一部分是创建一个 injector 对象并返回. 这个对象与之前 createInjector 的返回值是相同的:

```js
function createInternalInjector(cache, factoryFn) {
    // ...
    return {
        has: function(name) {
            return cache.hasOwnProperty(name) ||
            providerCache.hasOwnProperty(name + 'Provider');
        },
        get: getService,
        annotate: annotate,
        invoke: invoke,
        instantiate: instantiate
    };
}
```

The has method checks for the existence of a dependency in the local cache as
well as the provider cache.
Now that we have createInternalInjector, we can create our two injectors with it. The provider injector works with the provider cache. Its fallback function will throw an exception letting the
user know the dependency they’re looking for doesn’t exist:

get, invoke 和 instantiate 方法引用 createInternalInjector 中的函数. annotate 引用 annotate 函数. has

