### 注册一个模块

有了上面几节的基础, 让我们在本节进入实际想做的的事情: 注册模块.

本节所有在 loader_spec.js 中剩余的测试都将与模块加载器一起工作, 所以让我们创建一个嵌套的 describe 块, 并将模块加载器的设置放到一个 before 块中.

```js
describe('modules', function() {
    beforeEach(function() {
        setupModuleLoader(window);
    });
});
```

第一种我们将要测试的行为是, 调用 angular.module 方法并返回一个 module 对象.

angular.module 方法需要一个表示模块名称的字符串参数和一个可以为空数组的表示模块依赖的数组参数. 该方法构建一个 module 对象并返回. 这个 module 对象包含一个存有它的名称的 name 属性:

```js
it('allows registering a module', function() {
    var myModule = window.angular.module('myModule', []);
    expect(myModule).toBeDefned();
    expect(myModule.name).toEqual('myModule');
});
```

当你多次注册一个相同名称的 module, 新的 module 会取代旧的. 这也意味着使用相同的 name 两次滴啊用 module 方法, 会得到不同的 module 对象:

```js
it('replaces a module when registered with same name again', function() {
    var myModule = window.angular.module('myModule', []);
    var myNewModule = window.angular.module('myModule', []);
    expect(myNewModule).not.toBe(myModule);
});
```

在我们的 module 方法中, 让我们把创建 module 的工作交给一个叫 createModule 函数去做. 在这个函数中, 我们创建一个 module 对象并返回.

```js
function setupModuleLoader(window) {
    var ensure = function(obj, name, factory) {
        return obj[name] || (obj[name] = factory());
    };

    var angular = ensure(window, 'angular', Object);

    var createModule = function(name, requires) {
        var moduleInstance = {
            name: name
        };
        return moduleInstance;
    };

    ensure(angular, 'module', function() {
        return function(name, requires) {
            return createModule(name, requires);
        };
    });
}
```

除了 module 的名称, module 对象还应该含有一个模块依赖数组的引用:

```js
it('attaches the requires array to the registered module', function() {
    var myModule = window.angular.module('myModule', ['myOtherModule']);
    expect(myModule.requires).toEqual(['myOtherModule']);
});
```

我们给 module 对象添加一个模块依赖数组去满足这个需求:

```js
var createModule = function(name, requires) {
    var moduleInstance = {
        name: name,
        requires: requires
    };
    return moduleInstance;
};
```
