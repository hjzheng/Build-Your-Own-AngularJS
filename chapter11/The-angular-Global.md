### 全局对象 angular

如果你曾经使用过 Angular, 你或许对全局 angular 对象有兴趣. 这是我们在这里介绍全局 angular 对象的目的.

现在我们需要这个对象的原因是用来存储注册的 Angular 模块信息. 当我们开始创建模块和 injector 时, 我们需要存储它们.

在一个叫做 loader.js 的文件中, 实现处理模块的框架组件: 模块加载器. 我们在这里引入全局 angular 对象. 但是首先, 让我们添加一个测试.

```js
'use strict';
var setupModuleLoader = require('../src/loader');
describe('setupModuleLoader', function() {
    it('exposes angular on the window', function() {
        setupModuleLoader(window);
        expect(window.angular).toBeDefned();
    });
});
```

该测试假设已经有一个叫做 setupModuleLoader 的函数可以使用, 也假设你可以使用 window 对象作为参数调用 setupModuleLoader 函数. 当调用结束, window 对象上会有一个 angular 属性.

让我们创建一个 load.js 并使上面的测试通过:

```js
function setupModuleLoader(window) {
    var angular = window.angular = {};
}
module.exports = setupModuleLoader;
```
