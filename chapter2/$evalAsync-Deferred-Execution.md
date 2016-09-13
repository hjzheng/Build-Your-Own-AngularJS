### $evalAsync - 推迟代码的执行

在 JavaScript 的世界中，存在一个很常见的举动：推迟一段代码的执行，也就是说，将该段代码的执行推迟到将来的某个时间点，而这个时间点正是我们当前逻辑结束的时刻。最常见的做法是使用0 或者很短的时间间隔来调用 setTimeout()；

这种方法在 angular 中也是适用的，很多人偏爱使用 $timeout service 来实现延迟的目的，除此之外， $timeout service 还将代码的延迟执行与脏检查的过程通过 $apply 整合在了一起。

但是，angular 中还存在另一种延迟代码执行的方法，就是 Scope 对象拥有的 $evalAsync 方法。它接收一个 function 作为参数并安排其在正在进行的或者即将进行的脏检查过程中被执行。比如，你可以在一个监听器中推迟一段代码的执行，你会发现，即使该段代码被推迟执行了，它仍然会在当前的脏检查中被调用。使用 $evalAsync 的方式通常是比通过 0 调用 $timeout 更为可取的，其原因在于浏览器的 event loop 机制。

当你使用 $timeout 来安排一段代码的执行，意味着你将该段代码的执行控制交给了浏览器，让浏览器来决定什么时间点来执行该段代码。然而，在执行你的安排之前，浏览器可能会选择执行其他的工作，诸如渲染界面，执行点击事件以及 Ajax 回调操作。
相比较，$evalAsync 方法对延迟任务在什么时间点执行控制得更为精确， 通过其安排的延迟任务始终会在脏检查的过程中被执行，在浏览器选择执行其他工作执行，它保障了延迟任务的执行。

当你想要避免不必要的渲染时，$timeout 与 $evalAsync 这两种方式的区别就显得十分有意义了。当 DOM 的改变不稳定，立刻会被覆盖时，为什么还要浏览器去渲染这些改变呢，显然是没有意义的。

下面是一个关于 $evalAsync 的单元测试：
```js
// test/scope_spec.js
describe('$evalAsync', function() {
    var scope;
    beforeEach(function() {
        scope = new Scope();
    });
    it('executes given function later in the same cycle', function() {
        scope.aValue = [1, 2, 3];
        scope.asyncEvaluated = false;
        scope.asyncEvaluatedImmediately = false;
        scope.$watch(
            function(scope) { return scope.aValue; },
            function(newValue, oldValue, scope) {
                scope.$evalAsync(function(scope) {
                    scope.asyncEvaluated = true;
                });
                scope.asyncEvaluatedImmediately = scope.asyncEvaluated;
            }
        );
        scope.$digest();
        expect(scope.asyncEvaluated).toBe(true);
        expect(scope.asyncEvaluatedImmediately).toBe(false);
    });
});
```
上面的测试中，我们在监听器中调用了 $evalAsync，然后验证该任务是否会在接下来的脏检查中被执行，并且是在该监听器执行完成之后。

要实现这一点，首先，需要一种方式来存储经过 $evalAsync 安排的任务，我们可以在 Scope 的构造函数中使用一个数组来实现:
```js
// src/scope.js
function Scope() {
    this.$$watchers = [];
    this.$$lastDirtyWatch = null;
    this.$$asyncQueue = [];
}
```
接着，我们来实现 $evalAsync，将需要延迟执行的任务添加到这个队列中：
```js
//src/scope.js
Scope.prototype.$evalAsync = function(expr) {
    this.$$asyncQueue.push({scope: this, expression: expr});
};
```
目前我们已经存储好了等待运行的任务，但是还没有真正的运行它们。在脏检查的过程中，我们将实现这一动作：
首先，我们需要取出队列中所有等待运行的任务并且通过 $eval 的方式执行它们：
```
// src/scope.js
Scope.prototype.$digest = function() {
    var ttl = 10;
    var dirty;
    this.$$lastDirtyWatch = null;
    do {
        while (this.$$asyncQueue.length) {
            var asyncTask = this.$$asyncQueue.shift();
            asyncTask.scope.$eval(asyncTask.expression);
        }
        dirty = this.$$digestOnce();
        if (dirty && !(ttl--)) {
            throw '10 digest iterations reached';
        }
    } while (dirty);
};
```
这样的实现过程保证了，当你在脏检查的过程中安排了一个延迟执行的任务，这个任务会在这个脏检查过程中被执行，这样正满足了我们的测试用例。