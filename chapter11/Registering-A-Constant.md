### 注册一个 Constant

我们将要实现 Angular 应用组件中的第一种类型 constant. 你可以使用 constant 为 Angular 模块注册一个简单的值, 例如, 数字, 字符串, 一个对象, 或一个函数.

在我们注册一个 constant 并且创建一个 injector 之后, 我们可以使用 injector 的 has 方法检查 constant 是否存在:

```js
it('has a constant that has been registered to a module', function() {
    var module = window.angular.module('myModule', []);
    module.constant('aConstant', 42);
    var injector = createInjector(['myModule']);
    expect(injector.has('aConstant')).toBe(true);
});
```

在这里, 我们第一次看到定义一个 module 并为其创建一个 injector 的完整顺序. 关于创建 injector 的一个非常有意思的现象, 我们并没有传递给 createInjector 函数一个模块对象的引用, 而是一个模块名称, 并且期望可以在 angular.module 中找到它们:

作为一个检查, 让我们确保组件没有被注册时, has 方法返回 false:

```js
it('does not have a non-registered constant', function() {
    var module = window.angular.module('myModule', []);
    var injector = createInjector(['myModule']);
    expect(injector.has('aConstant')).toBe(false);
});
```

因此, 一个注册在模块中的 constant 如何在 injector 中变的可用? 首先, 我们需要在 module 对象中添加 constant 的注册方法:

```js
var createModule = function(name, requires, modules) {
    if (name === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid module name';
    }

    var moduleInstance = {
        name: name,
        requires: requires,
        constant: function(key, value) {

        }
    };

    modules[name] = moduleInstance;
    return moduleInstance;
};
```

关于模块和 injector 的一个通用规则, 模块实际上不存储任何应用组件, 只存储创建应用组件的方法. 实际上 injector 才是将它们变成具体组件的地方.

模块应该保存一些任务的集合, 例如注册一个 constant, 当模块加载时, injector 才去执行它们. 这些任务的集合叫做 invoke queue.
每个模块都有一个 invoke queue, 当模块被 injector 加载时, injector 运行 invoke queue 中的任务.

现在, 我们将要定义一个数组作为 invoke queue. 每个在 queue 中的数组都有两项: 注册的应用组件类型和注册该组件的参数.

```js
[
    ['constant', ['aConstant', 42]]
]
```

invoke queue 被存储在一个叫做 _invokeQueue 的模块对象属性上, 由于在 constant 函数中, 我们要 push 一个 constant 定义到 invoke queue 中, 所以我们在 createModule 引入一个本地变量:

```js
var createModule = function(name, requires, modules) {
    if (name === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid module name';
    }
    var invokeQueue = [];
    var moduleInstance = {
        name: name,
        requires: requires,
        constant: function(key, value) {
            invokeQueue.push(['constant', [key, value]]);
        },
        _invokeQueue: invokeQueue
    };
    modules[name] = moduleInstance;
    return moduleInstance;
};
```

然后, 我们来创建 injector. 我们应该遍历传入模块名称, 查找相应的模块对象, 然后再遍历它们的 invoke queue:

```js
'use strict';
var _ = require('lodash');
function createInjector(modulesToLoad) {
    _.forEach(modulesToLoad, function(moduleName) {
        var module = window.angular.module(moduleName);
        _.forEach(module._invokeQueue, function(invokeArgs) {

        });
    });
    return {};
}
module.exports = createInjector;
```

在 injector 内, 我们有一些代码知道如何处理 invoke queue 保存的每一项. 我们把这些代码放到一个叫做 $provide 的对象里. 当我们遍历 invoke queue
中的项时, 我们根据每个 invocation 数组的第一项从 $provide 中查找方法(例如 'constant'), 然后用 invocation 数组的第二项作为参数调用方法:

```js
function createInjector(modulesToLoad) {
    var $provide = {
        constant: function(key, value) {

        }
    };

    _.forEach(modulesToLoad, function(moduleName) {
        var module = window.angular.module(moduleName);
        _.forEach(module._invokeQueue, function(invokeArgs) {
            var method = invokeArgs[0];
            var args = invokeArgs[1];
            $provide[method].apply($provide, args);
        });
    });

    return {};
}
```

所以, 当你在 module 上调用一个方法, 例如 constant 时, 这会引起 createInjector 方法中的 $provide 对象中相同参数的相同方法被调用, 但是它不会立即发生, 只有模块加载的时候. 与此同时, 关于方法调用的信息被存储在 invoke queue.

仍然还剩注册 constant 的逻辑没被实现. 不同的类型的应用组件需要不同的初始化逻辑, 但是它们有共同的地方, 一旦它们被初始化后, 它们会被 injector 保存. constant 实际上是非常简单的东西, 我们可以把它放在 cache 中. 然后我们实现 injector 的 has 方法, 用来在 cache 中检查相对应的 key.

```js
function createInjector(modulesToLoad) {
    var cache = {};
    var $provide = {
        constant: function(key, value) {
            cache[key] = value;
        }
    };

    _.forEach(modulesToLoad, function(moduleName) {
        var module = window.angular.module(moduleName);
        _.forEach(module._invokeQueue, function(invokeArgs) {
            var method = invokeArgs[0];
            var args = invokeArgs[1];
            $provide[method].apply($provide, args);
        });
    });

    return {
        has: function(key) {
            return cache.hasOwnProperty(key);
        }
    };
}
```

现在, 我们又一次遇到一种情况: 需要保护对象的 hasOwnProperty 属性. 不允许注册一个叫做 hasOwnProperty 的 constant:

```js
it('does not allow a constant called hasOwnProperty', function() {
    var module = window.angular.module('myModule', []);
    module.constant('hasOwnProperty', false);

    expect(function() {
        createInjector(['myModule']);
    }).toThrow();

});
```

除此之外检查一个应用组件是否存在方法, injector 也要有一个获取组件的本身的方法. 基于此, 我们引入一个叫做 get 的方法:

```js
it('can return a registered constant', function() {
    var module = window.angular.module('myModule', []);
    module.constant('aConstant', 42);
    var injector = createInjector(['myModule']);
    expect(injector.get('aConstant')).toBe(42);
});
```

该方法简单的通过 key 从缓存中查找:

```js
return {
    has: function(key) {
        return cache.hasOwnProperty(key);
    },
    get: function(key) {
        return cache[key];
    }
};
```
