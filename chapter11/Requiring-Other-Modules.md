### 依赖其他模块

到目前为止, 我们已经通过加载单个模块创建 injector, 但是也可以通过加载多个模块创建 injector. 最直接的方式是将多个模块名称放到一个数组中提供给 createInjector, 所有给定模块的应用组件都将被注册:

```js
it('loads multiple modules', function() {
    var module1 = window.angular.module('myModule', []);
    var module2 = window.angular.module('myOtherModule', []);
    module1.constant('aConstant', 42);
    module2.constant('anotherConstant', 43);
    var injector = createInjector(['myModule', 'myOtherModule']);
    expect(injector.has('aConstant')).toBe(true);
    expect(injector.has('anotherConstant')).toBe(true);
});
```

这个我们已经处理过了, 因为我们在 createInjector 中遍历 modulesToLoad 数组.
另一种引起多个模块被加载的方式是加载的模块依赖其他模块. 当使用 angular.module 方法注册一个模块时, 会有第二个数组参数, 目前是一个空数组, 但是它也可以存放依赖的模块名称. 当模块加载时, 这些依赖的模块也会被加载.

```js
it('loads the required modules of a module', function() {
    var module1 = window.angular.module('myModule', []);
    var module2 = window.angular.module('myOtherModule', ['myModule']);

    module1.constant('aConstant', 42);
    module2.constant('anotherConstant', 43);

    var injector = createInjector(['myOtherModule']);

    expect(injector.has('aConstant')).toBe(true);
    expect(injector.has('anotherConstant')).toBe(true);
});
```

```js
it('loads the transitively required modules of a module', function() {
    var module1 = window.angular.module('myModule', []);
    var module2 = window.angular.module('myOtherModule', ['myModule']);
    var module3 = window.angular.module('myThirdModule', ['myOtherModule']);

    module1.constant('aConstant', 42);
    module2.constant('anotherConstant', 43);
    module3.constant('aThirdConstant', 44);

    var injector = createInjector(['myThirdModule']);

    expect(injector.has('aConstant')).toBe(true);
    expect(injector.has('anotherConstant')).toBe(true);
    expect(injector.has('aThirdConstant')).toBe(true);
});
```

这种工作方式其实很简单, 当我们加载一个模块, 在遍历模块的 invoke queue 之前, 我们遍历该模块依赖的模块, 递归的加载它们每一个. 因此我们需要一个模块加载函数, 可以递归调用它:

```js
_.forEach(modulesToLoad, function loadModule(moduleName) {
    var module = window.angular.module(moduleName);
    _.forEach(module.requires, loadModule);
    _.forEach(module._invokeQueue, function(invokeArgs) {
            var method = invokeArgs[0];
            var args = invokeArgs[1];
            $provide[method].apply($provide, args);
    });
});
```

当你有模块依赖其他模块时, 它很容易出现一种情况, 在多个模块中循环依赖:

```js
it('loads each module only once', function() {
    window.angular.module('myModule', ['myOtherModule']);
    window.angular.module('myOtherModule', ['myModule']);
    createInjector(['myModule']);
});
```

我们当前的实现在加载模块时, 总是递归也不检查模块是否需要加载.

通过确保每个模块只被加载一次来处理循环依赖. 这样做还有一个效果是当有两个相同的模块要被加载, 不存在循环, 它不会被加载两次, 因此避免了额外的工作.
我们将引入一个对象, 追踪模块已经被加载过. 在加载一个模块前, 检查该模块是否已经被加载过:


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
        }
    };
}
```
