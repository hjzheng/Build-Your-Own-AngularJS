### 放弃不稳定的 Digest

当前我们的实现, 存在一个很明显的漏洞, 就是如果存在两个监听器 (watcher) 相互监控彼此产生的改变, 也就是说, 数据永远都是脏的, 永远都不稳定. 下面的测试用例展示了这种场景:

```js
it('gives up on the watches after 10 iterations', function() {
  scope.counterA = 0;
  scope.counterB = 0;
  scope.$watch(
        function(scope) { return scope.counterA; },
        function(newValue, oldValue, scope) {
          scope.counterB++;
        }
  );
  scope.$watch(
        function(scope) { return scope.counterB; },
        function(newValue, oldValue, scope) {
          scope.counterA++;
        }
  );
  expect((function() { scope.$digest(); })).toThrow();
});
```

我们期望 scope.$digest 方法抛出一个异常, 但是它永远不会. 实际上, 该测试永远不会结束, 那是因为, 两个counter变量 (counterA 和 counterB) 相互彼此依赖, 在每次遍历的时候, 它们中的一个 $$digestOnce 方法肯定返回 true, 也是就是数据始终是脏的.

我们需要做的是保持运行 digest 在一个可接受的次数内, 如果运行超过这个次数, 数据仍然还在改变, 抛出一个异常, 表示它是不稳定的.

这个可接受次数被称作 TTL (Time To Live) , 默认是10次, 看起来有些小, 但是, 记住, 这是一个性能敏感区, 因为 digest 经常发生, 并且每个 digest 要运行所有的 watch 函数. 用户也不太可能创建 10 个以上链式监听.

让我们继续, 在外层 digest 循环添加一个循环计数器, 如果到达 TTL, 抛出一个异常:


```js
Scope.prototype.$digest = function() {
    var ttl = 10;
    var dirty;
    do {
        dirty = this.$$digestOnce();
        if (dirty && !(ttl--)) {
            throw '10 digest iterations reached';
        }
    } while (dirty);
};
```

这个更新过的版本会引起我们的测试用例抛出异常, 这正是我们所期望的.