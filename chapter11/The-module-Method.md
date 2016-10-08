### module 方法

我们将介绍 angular 的第一个方法, 也是我们将会在下面的章节中使用很多的方法: module. 我们判断该方法已经存在于刚创建的 angular 全局对象中:

```js
it('exposes the angular module function', function() {
    setupModuleLoader(window);
    expect(window.angular.module).toBeDefned();
});
```

和 angular 全局对象一样, 当 setupModuleLoader 方法被调用多次时, module 方法不应该被覆盖:

```js
it('exposes the angular module function just once', function() {
    setupModuleLoader(window);
    var module = window.angular.module;
    setupModuleLoader(window);
    expect(window.angular.module).toBe(module);
});
```

我们可以重用 ensure 函数去构建 module 方法.

```js
function setupModuleLoader(window) {
    var ensure = function(obj, name, factory) {
        return obj[name] || (obj[name] = factory());
    };

    var angular = ensure(window, 'angular', Object);

    ensure(angular, 'module', function() {
        return function() {

        };
    });
}
```
