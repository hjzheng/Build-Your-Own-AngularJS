### Injector

让我们调整一下方向, 来说说 Angular 依赖注入的基础: injector.

injector 不是模块加载器(module loader)的一部分, 是一个独立服务, 所以我们将代码和测试放在新文件里. 在这些测试中, 我们将假设一个新的模块加载已经设置完成:

```js
'use strict';
var setupModuleLoader = require('../src/loader');

describe('injector', function() {
    beforeEach(function() {
        delete window.angular;
        setupModuleLoader(window);
    });
});
```

通过调用函数 createInjector 创建一个 injector. createInjector 函数需要一个 module 名称的数组参数, 并返回一个 injector 对象:

```js
'use strict';
var setupModuleLoader = require('../src/loader');
var createInjector = require('../src/injector');

describe('injector', function() {
    beforeEach(function() {
        delete window.angular;
        setupModuleLoader(window);
    });

    it('can be created', function() {
        var injector = createInjector([]);
        expect(injector).toBeDefned();
    });
});
```

现在, 我们先给出一个实现, 简单返回一个空对象字面量:

```js
'use strict';
function createInjector(modulesToLoad) {
    return {};
}
module.exports = createInjector;
```
