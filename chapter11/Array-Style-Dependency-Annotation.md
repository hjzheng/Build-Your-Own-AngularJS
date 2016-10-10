### 数组风格的依赖标注

虽然你可以一直使用函数的 $inject 属性标示注入函数的依赖, 你不可能总想这样做, 因此这种方式有些繁琐.
一种稍微不繁琐的提供依赖名称的方式是为 injector.invoke 提供一个数组而不是一个函数. 在这个数组中, 首先给出依赖的名称, 数组的最后一项是实际被调用的函数:

```js
['a', 'b', function(one, two) {
    return one + two;
}]
```
因为我们现在谈论多种不同方式去标示函数依赖, 所以调用一个方法用来提取任何类型标示的函数依赖. 这样的一个方法实际上是在 injector 中实现和公开的, 它叫做 annotate.

```js
describe('annotate', function() {
    it('returns the $inject annotation of a function when it has one', function() {
        var injector = createInjector([]);

        var fn = function() { };

        fn.$inject = ['a', 'b'];

        expect(injector.annotate(fn)).toEqual(['a', 'b']);
    });
});
```

我们将在 createInjector 中引入一个本地函数, 作为一个 injector 的一个属性公开出去:

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

    function annotate(fn) {
        return fn.$inject;
    }

    function invoke(fn, self, locals) {
        var args = _.map(fn.$inject, function(token) {
            if (_.isString(token)) {
                return locals && locals.hasOwnProperty(token) ?
                locals[token] :
                cache[token];
            } else {
                throw 'Incorrect injection token! Expected a string, got '+token;
            }
        });

        return fn.apply(self, args);
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
        annotate: annotate,
        invoke: invoke
    };
}
```

基于数组风格依赖注入, 当传入一个数组时, annotate 从数组中提取依赖名称:

```js
it('returns the array-style annotations of a function', function() {
    var injector = createInjector([]);
    var fn = ['a', 'b', function() { }];

    expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
```

如果 fn 是一个数组, annotate 方法应该返回除了最后一项的数组:

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else {
        return fn.$inject;
    }
}
```
