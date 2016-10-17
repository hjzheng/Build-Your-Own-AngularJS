### 循环依赖

因为我们现在有依赖依赖于其他依赖, 我们介绍一种可能发生的情况, 循环依赖链: 如果 A 依赖于 B, B 依赖于 C 并且 C 又依赖于 A. 没有一种方式可以在构造 A, B 或 C 时避免进入无限循环. 或许我们可以显示一些有用的错误信息.

```js
it('notifes the user about a circular dependency', function() {
    var module = window.angular.module('myModule', []);

    module.provider('a', {$get: function(b) { }});
    module.provider('b', {$get: function(c) { }});
    module.provider('c', {$get: function(a) { }});

    var injector = createInjector(['myModule']);
    expect(function() {
        injector.get('a');
    }).toThrowError(/Circular dependency found/);

});
```

这个小技巧在这儿有两部分: 第一, 在调用 $get 方法构建依赖之前, 我们在 instanceCache 中放一个特殊标记值. 这个标记值告诉我们, 依赖正在被构建.
第二, 当我们查找依赖时, 发现了这个标记, 就意味着, 我们正在试图查找正在构建的依赖, 也就是说存在循环依赖.

我们可以使用一个空对象作为标记值. 非常重要的事情是它不等于任何其他事物. 让我们在 injector.js 开头引入这个标记:

```js
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
var INSTANTIATING = { };
```

在 getService 方法中, 在调用 provider 之前, 我们将这个标记放入 instanceCache 中, 当查找实例时, 检查它:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        if (instanceCache[name] === INSTANTIATING) {
            throw new Error('Circular dependency found');
        }
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        instanceCache[name] = INSTANTIATING;
        var provider = providerCache[name + 'Provider'];
        var instance = instanceCache[name] = invoke(provider.$get);
        return instance;
    }
}
```

因为依赖最终是会被构建的, 所以 INSTANTIATING 标志在 instanceCache 中会被替换的. 但是在实例化时, 一些事情也会出错, 此时我们不想将标记留在 instanceCache 中:

```js
it('cleans up the circular marker when instantiation fails', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', {$get: function() {
        throw 'Failing instantiation!';
    }});

    var injector = createInjector(['myModule']);

    expect(function() {
        injector.get('a');
    }).toThrow('Failing instantiation!');

    expect(function() {
        injector.get('a');
    }).toThrow('Failing instantiation!');

});
```

我们正在这里做的就是尝试初始化实例并失败两次, 它将尝试调用 provider 两次. 当前的实现不会这样做, 因为在第一次, 在 instanceCache 中留下了
INSTANTIATING 标记, 在第二次调用时, 发现 INSTANTIATING 标志, 得出这是一个循环依赖的结论. 所以, 我们要确保如果实例化依赖失败, 不要留下 INSTANTIATING 标记:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        if (instanceCache[name] === INSTANTIATING) {
            throw new Error('Circular dependency found');
        }
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        instanceCache[name] = INSTANTIATING;
        try {
            var provider = providerCache[name + 'Provider'];
            var instance = instanceCache[name] = invoke(provider.$get);
            return instance;
        } fnally {
            if (instanceCache[name] === INSTANTIATING) {
                delete instanceCache[name];
            }
        }
    }
}
```

只是通知用户有一个循环依赖, 不是非常的有用. 最好可以让他们知道问题出在哪里:

我们想要显示用户发生问题的依赖关系路径. 在我们例子中, 请从右往左读:

```a <- c <- b <- a```

更新现有的测试用例, 并期望一个这样的错误信息:

```js
it('notifes the user about a circular dependency', function() {
    var module = window.angular.module('myModule', []);
    module.provider('a', {$get: function(b) { }});
    module.provider('b', {$get: function(c) { }});
    module.provider('c', {$get: function(a) { }});
    var injector = createInjector(['myModule']);
    expect(function() {
        injector.get('a');
    }).toThrowError('Circular dependency found: a <- c <- b <- a');
});
```

我们需要做的是用一种数据结构存储当前的依赖关系路径. 让我们在 createInjector 方法中引入一个数组:

```js
function createInjector(modulesToLoad, strictDi) {
var providerCache = {};
var instanceCache = {};
var loadedModules = {};
var path = [];
// ...
```

In getService() we can treat path essentially as a stack. When we start resolving a dependency,
we add its name to the front of the path. When we’re done, we pop it off:

在 getService 方法中, 本质上, 我们可以将路径看做一个堆栈. 当我们开始分析一个依赖, 我们将它的名字添加到 path 的前面. 当我们完成时, 弹出它:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        if (instanceCache[name] === INSTANTIATING) {
            throw new Error('Circular dependency found');
        }
        return instanceCache[name];
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

如果, 我们遇到一个循环依赖, 我们可以使用当前的 path 值来显示有问题的用户依赖关系路径:

```js
function getService(name) {
    if (instanceCache.hasOwnProperty(name)) {
        if (instanceCache[name] === INSTANTIATING) {
            throw new Error('Circular dependency found: ' +
            name + ' <- ' + path.join(' <- '));
        }
        return instanceCache[name];
    } else if (providerCache.hasOwnProperty(name + 'Provider')) {
        path.unshift(name);
        instanceCache[name] = INSTANTIATING;
        try {
            var provider = providerCache[name + 'Provider'];
            var instance = instanceCache[name] = invoke(provider.$get, provider);
            return instance;
        } finally {
            path.shift();
        }
    }
}
```

现在, 用户可以看出哪里有问题了!
