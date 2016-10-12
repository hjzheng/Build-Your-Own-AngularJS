### 将 annotate 集成到 invoke

现在, 我们可以使用三种 Angular 支持的不同的方法去获取依赖的名称, 三种方法分别是 $inject, 数组包裹和从函数源码中抽取.
我们仍然需要做的是将依赖名称的查找集成到 injector.invoke 方法中. 你应该能够给它一个数组标注的函数, 并期望它做正确的事情:

```js
it('invokes an array-annotated function with dependency injection', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);
    var injector = createInjector(['myModule']);
    var fn = ['a', 'b', function(one, two) { return one + two; }];
    expect(injector.invoke(fn)).toBe(3);
});
```

用完全相同的方式, 你应该能够给它一个未标注的函数, 并期望它从源码中解析依赖标注:

```js
it('invokes a non-annotated function with dependency injection', function() {
    var module = window.angular.module('myModule', []);
    module.constant('a', 1);
    module.constant('b', 2);
    var injector = createInjector(['myModule']);
    var fn = function(a, b) { return a + b; };
    expect(injector.invoke(fn)).toBe(3);
});
```

在 invoke 方法中, 我们需要做两件事: 第一件, 需要使用 annotate 方法替代直接访问 $inject 查找依赖名称. 第二件, 我们需要检查, 如果这个函数被一个数组包裹着, 在调用它之前, 需要去掉包裹的数组:

```js
function invoke(fn, self, locals) {
    var args = _.map(annotate(fn), function(token) {
        if (_.isString(token)) {
            return locals && locals.hasOwnProperty(token) ? locals[token] : cache[token];
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

现在, 我们提供这三种类型的函数调用了.
