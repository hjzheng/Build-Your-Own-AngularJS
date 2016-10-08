### 初始化 angular 全局对象, 只能初始化一次

因为 angular 全局对象提供存储注册模块的功能, 它本质上是一个全局状态持有者. 这意味着我们需要一些措施去管理这些状态. 首先, 我们想为每一个测试用例提供一个干净的开始, 所以在每个测试开头我们需要清除任何已经存在的 angular 全局对象.

```js
beforeEach(function() {
    delete window.angular;
});
```

同样的, 在 setupModuleLoader 中, 我们必须要小心, 不要覆盖了已经存在的 angular 全局对象. 即使 setupModuleLoader 被调用多次. 当你调用相同的 window 对象作为参数调用 setupModuleLoader 两次, 调用参数的 angular 全局对象应该指向完全相同的对象.

```js
function setupModuleLoader(window) {
    var angular = (window.angular = window.angular || {});
}
```

我们很快还会使用这种 "load once" 模式, 所以, 我们将它抽象成一个叫做 ensure 的通用函数. 该函数需要三个参数: 一个对象, 一个属性名, 一个工厂方法. 该函数使用工厂方法生成一个属性, 仅当该属性不存在.

```js
function setupModuleLoader(window) {
    var ensure = function(obj, name, factory) {
        return obj[name] || (obj[name] = factory());
    };

    var angular = ensure(window, 'angular', Object);
}
```

在这种情况下, 我们调用 Object() 方法赋给 window.angular 一个空对象, 实际上这和调用 new Object() 一样.
