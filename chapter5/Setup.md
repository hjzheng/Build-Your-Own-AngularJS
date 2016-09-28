### 准备

我们仍然将继续在 Scope 对象上工作, 所以所有的代码都将写到 src/scope.js 和 test/scope_spec.js 中. 让我们开始吧!

```js
describe('Events', function() {
    var parent;
    var scope;
    var child;
    var isolatedChild;
    beforeEach(function() {
        parent = new Scope();
        scope = parent.$new();
        child = scope.$new();
        isolatedChild = scope.$new(true);
    });
});
```

关于这次实现, 我们有好多与 Scope 继承层级相关的事情要做, 因此为了方便, 我们在 beforeEach 函数中做了些配置: 一个父 Scope, 两个子 Scope, 其中一个是隔离 Scope 的. 这应当可以覆盖关于继承所需要的测试了.

