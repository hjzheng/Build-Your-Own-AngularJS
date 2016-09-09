### Scope 对象

Scope 对象是通过构造函数创建的, 它就是一个 POJO (简单的 JavaScript 对象). 让我们来写一个测试用例.

创建一个 `test/scope_spec.js` 文件, 并添加如下内容:

```js
'use strict';
var Scope = require('../src/scope');

describe('Scope', function() {

  it('can be constructed and used as an object', function() {
    var scope = new Scope();
    scope.aProperty = 1;

    expect(scope.aProperty).toBe(1);
  });
});
```
测试文件最上方启用 ES5 严格模式, 然后通过 requireJS 方式引入 Scope, 使用该 Scope 创建一个 Scope 对象, 为 Scope 对象随意添加一个属性, 并检查它是否存在.

该测试用例很容易通过, 我们来创建一个 `src/scope.js`, 并添加如下内容

```js
'use strict';
function Scope() {

}
module.exports = Scope;
```

在本测试用例中, 我们在 Scope 对象上添加一个 aProperty 属性. 属性在 scope 上如何工作是非常明确的, 就是简单的 JavaScript 属性, 并没有任何特别之处, 当你为其赋值时, 既不需要调用特殊的 setter 方法, 也不必对值做任何限制.
相反的, 在两个非常特殊的函数 `$watch` 和 `$digest` 中会发生一些神奇的事情, 我们来关注他们.