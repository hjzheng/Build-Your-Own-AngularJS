### 依赖的延迟实例化

现在, 我们可以在依赖之间使用依赖, 我们将要开始考虑这些事情完成的顺序. 当我们创建一个依赖的时候, 我们所拥有的所有的依赖都可用吗? 考虑下面的情况, b 依赖 a, 但是 b 注册在 a 之前:

```js
it('injects the $get method of a provider lazily', function() {
    var module = window.angular.module('myModule', []);
    module.provider('b', {
        $get: function(a) {
            return a + 2;
        }
    });

    module.provider('a', {$get: _.constant(1)});

    var injector = createInjector(['myModule']);

    expect(injector.get('b')).toBe(3);
});
```

这个测试的失败是因为我们尝试为 b 调用 $get 方法, 它的依赖的 a 还不可用.

如果真实 Angular injector 是这样工作的, 你必须非常小心按照依赖顺序加载你的代码. 幸运的是, injector 不是这样工作的. 相反的, injecter 会延迟调用这些 $get 方法, 只有当返回值是需要的时候.
因此, 即使 b 第一个被注册, 这时它的 $get 方法也不会被调用. 只有当我们要求 injector b 它才被调用, 这个时候 a 已经被注册过了.

因为, 当我们遍历 invoke queue 时, 此时不能调用 provider 的 $get, 我们需要保持住 provider 对象稍后调用它. 我们有一个用于存储的 cache 对象, 但是将 provider 对象放在这里是讲不通的, 因为 cache 是用来存储依赖实例的, 不是用来存储生产它们的 provider 的. 我们需要将 cache 分为两个: 一个存储所有实例, 一个存储所有 provider

```js
function createInjector(modulesToLoad, strictDi) {
    var providerCache = {};
    var instanceCache = {};
    var loadedModules = {};
    // ...
}
```

然后, 在 $provider 中, 我们把 constant 放入 instanceCache, 把 provider 放入 providerCache 中. 当存储一个 provider 时, 我们在 cache 的 key 上加一个 provider 前缀. 因此, 注册的 a provider, 它的会以 'aProvider' 作为 key 存入 provider 的 cache 中. 我们需要一个清楚的界限, 在 instance 和它们的 provider 之间, 它们的名字强化这种区别:

```js
var $provide = {
    constant: function(key, value) {
        if (key === 'hasOwnProperty') {
            throw 'hasOwnProperty is not a valid constant name!';
        }
        instanceCache[key] = value;
    },

    provider: function(key, provider) {
        providerCache[key + 'Provider'] = provider;
    }
};
```

让我们创建一个 getService 方法, 给一个依赖名称参数, 首先, 在 instanceCache 中查找, 如果有, 直接返回, 反之, 在 providerCache 中查找, 如果找到, 实例化并返回:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        var provider = providerCache[name + 'Provider'];
        return invoke(provider.$get, provider);
    }
}
```

在 invoke 的依赖查找循环中, 可以调用我们的新函数 getService, 如果从本地查找失败, 我们可以根据依赖名称在 cache 中查找:


```js
function invoke(fn, self, locals) {
    var args = _.map(annotate(fn), function(token) {

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
```

最后, 我们也需要更新 injector.get 和 injector.has 方法, 让它们以新方式工作.  injector.get 方法只是 getService 方法的一个别名. 在 has 方法中, 我们需要对 instanceCache 和 providerCache 做一个属性检查, 当检查 providerCache 时, 加上 Provider 后缀:

```js
return {
    has: function(key) {
        return instanceCache.hasOwnProperty(key) ||
            providerCache.hasOwnProperty(key + 'Provider');
    },
    get: getService,
    annotate: annotate,
    invoke: invoke
};
```

所以, 一个 provider 的依赖, 只有当它被注入到某个地方时被实例化, 或者通过 injector.get 被显示调用. 如果没有人请求依赖, 依赖将永远不会被实例化, 也就是说它的 provider 的 $get 永远不会调用.

你通过 injector.has 可以检查依赖是否存在, 这样做, 并不会引起依赖被实例化. 它仅仅检查依赖的实例或 provider 是否可用.
