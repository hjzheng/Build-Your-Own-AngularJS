### 监控对象的属性: `$watch` 和 `$digest`

`$watch` 和 `$digest` 就像一枚硬币的两面, 共同组成了 AngularJS 的核心, 脏检查循环 (digest cycle): 对数据变化的响应.

你可以通过 $watch 方法在 scope 上添加一个监听器 (watcher), 当 Scope 中发生一些变化的时候, 监听器会收到通知. $watch 方法需要提供两个函数, 才可以创建一个监听器:

- watch 函数, 指定你感兴趣的数据.
- listener 函数, 当数据发生变化时, 该函数会被调用.

硬币的另一面: `$digest` 函数, 它会遍历你添加到 scope 上的所有监听器 (watcher), 并运行相对应的 listener 函数.

让我们来详细说明这些基础, 先定义一个测试用例: 使用 `$watch` 方法注册一个监听器 (watcher), 当某人调用 `$digest` 方法时, 监听器的 listener 函数会被执行.

为了让事情变得更容易管理, 在 `test/scope_spec.js` 添加一个嵌套的 describe 块. 并且创建一个 beforeEach 函数用来初始化 scope, 这样就不用在每次测试前重复初始化 scope.

```js
describe("Scope", function() {

    it("can be constructed and used as an object", function() {
        var scope = new Scope();
        scope.aProperty = 1;
        expect(scope.aProperty).toBe(1);
    });

    describe("digest", function() {
        var scope;
        beforeEach(function() {
            scope = new Scope();
        });
        it("calls the listener function of a watch on first $digest", function() {
            var watchFn = function() { return 'wat'; };
            var listenerFn = jasmine.createSpy();
            scope.$watch(watchFn, listenerFn);

            scope.$digest();

            expect(listenerFn).toHaveBeenCalled();
        });
    });
});

```

在本测试中, 调用 `$watch` 方法在 scope 上注册一个监听器 (watcher), 我们并不关心 watch 函数, 因此仅仅让它返回一个不变的值. 再提供一个 Jasmine Spy 做为 listener 函数. 调用 $digest 方法, 然后检查 listener 函数是否被调用.

为了使该测试通过, 我们需要做一些事情. 首先, Scope 上需要有一个地方用来存储所有被注册的监听器, 让我们为 Scope 的构造函数添加一个数组吧:

```js
function Scope() {
    this.$$watchers = [];
}
```

现在, 我们可以定义 `$watch` 方法了. 该方法需要两个函数作为参数, 并将它们存储到 `$$watchers` 数组中, 我们想让每一个 Scope 对象都拥有该函数, 因此将它添加到 Scope 的原型上:

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
    var watcher = {
        watchFn: watchFn,
        listenerFn: listenerFn
    };
    this.$$watchers.push(watcher);
};
```
最后到了 `$digest` 函数, 我们目前先来定义一个简单版本的 `$digest` 函数吧, 仅仅提供遍历并执行所有注册的监听器 (watcher) 功能:

```js
Scope.prototype.$digest = function() {
    _.forEach(this.$$watchers, function(watcher) {
        watcher.listenerFn();
    });
};
```
`$degist` 函数使用了 LoDash 库中的 forEach 函数, 因此我们需要在文件上面引入 LoDash 库:

```js
'use strict'

var _ = require('lodash');
```
测试通过了, 但是这个版本的 `$degist` 不是很有用. 我们真正想要的是通过 watch 函数指定的值发生了变化时, 各自的 listener 函数会被执行. 这被称为脏检查. (dirty-checking)
