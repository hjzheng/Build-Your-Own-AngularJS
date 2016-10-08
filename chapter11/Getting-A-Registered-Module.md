### 获取一个注册的模块

angular.module 提供的另一个行为就是回去一个早前已经注册的 module 对象. 这个你可以通过省略第二参数去做.

```js
it('allows getting a module', function() {
    var myModule = window.angular.module('myModule', []);
    var gotModule = window.angular.module('myModule');
    expect(gotModule).toBeDefned();
    expect(gotModule).toBe(myModule);
});
```

我们引入另一个私有函数去获取 module, 叫 getModule. 我们也需要一个存放已经注册的 module 对象的地方. 我们在 ensure 函数中添加一个私有对象.
将私有变量作为参数传递给 createModule 和 getModule 方法:

```js
ensure(angular, 'module', function() {
    var modules = {};
    return function(name, requires) {
        if (requires) {
            return createModule(name, requires, modules);
        } else {
            return getModule(name, modules);
        }
    };
});
```

在 createModule 中我们必须存储新创建的 module 对象:

```js
var createModule = function(name, requires, modules) {
    var moduleInstance = {
        name: name,
        requires: requires
    };
    modules[name] = moduleInstance;
    return moduleInstance;
};
```

在 getModule 中, 我们查找 module 对象:

```js
var getModule = function(name, modules) {
    return modules[name];
};
```

基本上我们将所有的注册过的 module 保留在本地 modules 变量上. 这就是为什么只定义一次 angular 和 angular.module 是如此重要. 否则本地变量会被清除.

现在, 当你尝试获得一个不存在的 module, 你会获得一个 undefined 作为一个返回值. 应该用一个异常代替 undefined.


```js
it('throws when trying to get a nonexistent module', function() {
    expect(function() {
        window.angular.module('myModule');
    }).toThrow();
});
```

在 getModule 中我们应该尝试返回 module 对象之前, 检查是否存在:

```js
var getModule = function(name, modules) {
    if (modules.hasOwnProperty(name)) {
        return modules[name];
    } else {
        throw 'Module '+name+' is not available!';
    }
};
```

最终, 我们使用 hasOwnProperty 方法去检查一个 module 是否存在, 我们必须小心不要在 module cache 上覆盖该方法. 所以不允许注册一个叫做 hasOwnProperty module.

```js
it('does not allow a module to be called hasOwnProperty', function() {
    expect(function() {
        window.angular.module('hasOwnProperty', []);
    }).toThrow();
});
```

这个检查在 createModule 中:

```js
var createModule = function(name, requires, modules) {
    if (name === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid module name';
    }
    var moduleInstance = {
        name: name,
        requires: requires
    };
    modules[name] = moduleInstance;
    return moduleInstance;
};
```
