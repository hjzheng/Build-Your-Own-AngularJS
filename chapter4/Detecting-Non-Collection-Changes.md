### 检测非集合数据的变化

虽然 $watchCollection 的目的是监控数组和对象, 但是当 watch 函数返回非集合数据时, 它同样也可以工作. 在非集合数据的情况下相当于退回到之前的实现, 只需要简单地调用一下 $watch.

这里有一个用来验证这种行为的测试:

```js
it('works like a normal watch for non-collections', function() {
    var valueProvided;
    scope.aValue = 42;
    scope.counter = 0;
    scope.$watchCollection(
        function(scope) { return scope.aValue; },
        function(newValue, oldValue, scope) {
            valueProvided = newValue;
            scope.counter++;
        }
    );
    scope.$digest();
    expect(scope.counter).toBe(1);
    expect(valueProvided).toBe(scope.aValue);
    scope.aValue = 43;
    scope.$digest();
    expect(scope.counter).toBe(2);
    scope.$digest();
    expect(scope.counter).toBe(2);
});
```

我们使用 $watchCollection 在 Scope 上监控一个数字. 在 listener 函数给 counter 变量做加 1, 并对一个本地变量赋新值. 然后我们断言这个监听器会用新值调用 listener 函数, 如同一个普通, 非集合的监听器做的一样.

在监听器调用期间, $watchCollection 首先调用原生的 watch 函数拿到我们想监控的值, 然后通过与之前的值比较来检查它的变化, 存储该值用于下次 digest 循环:

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var newValue;
    var oldValue;
    var internalWatchFn = function(scope) {
        newValue = watchFn(scope);
        // Check for changes
        oldValue = newValue;
    };
    var internalListenerFn = function() {
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```

通过在内部的 watch 函数外声明变量保存 newValue 和 oldValue, 这样我们就可以在内部的 watch 函数和 listener 函数之间共享它们. 另外, 通过 $watchCollection 函数这种闭包形式, 它们同样可以在 digest 循环之间持续存在.

这对旧值是非常重要的, 因为我们需要在不同 digest 循环中对它进行比较.

内部的 listener 函数仅仅封装一下原生的 listener 函数, 传入 newValue 和 oldValue 作为参数:

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var internalWatchFn = function(scope) {
        newValue = watchFn(scope);
        // Check for changes
        oldValue = newValue;
    };
    var internalListenerFn = function() {
        listenerFn(newValue, oldValue, self);
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```

回想 $digest 决定是否调用 listener 函数, 是通过比较 watch 函数连续返回的值. 在内部的 watch 函数, 当前并没有返回值, 因此 listener 函数将永远不会被调用.

那么, 内部的 watch 函数到底应该返回什么? Since nothing outside of $watchCollection will ever see it, it doesn’t make that much difference. The only important thing is that it is different between successive invocations if there have been changes. That is what will cause the listener to get called.

Angular 的实现方式是通过引入一个 integer 类型的变量 counter 并且每当检测到变化时进行自增操作. 每个通过 $watchCollection 注册的监听器都会拥有自己的 counter 变量, 然后通过内部的 watch 函数返回它. 我们要确保 watch 函数被执行.

在非集合情况下, 我们只比较新旧值的引用:

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var changeCount = 0;
    var internalWatchFn = function(scope) {
        newValue = watchFn(scope);
        if (newValue !== oldValue) {
            changeCount++;
        }
        oldValue = newValue;
        return changeCount;
    };
    var internalListenerFn = function() {
        listenerFn(newValue, oldValue, self);
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```

现在, 非集合测试通过了, 但是如果非集合测试中的值是 NaN 呢?

```js
it('works like a normal watch for NaNs', function() {
  scope.aValue = 0/0;
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  scope.$digest();
  expect(scope.counter).toBe(1);
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

该测试失败的原因和第一章 NaN 中的是一样的: NaN 不等于任何其他值. 因此不要使用 !== 进行比较, 而是使用已存在的帮助函数 $$areEqual, 因为它已经帮我们处理好了 NaN.

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var self = this;
    var newValue;
    var oldValue;
    var changeCount = 0;
    var internalWatchFn = function(scope) {
        newValue = watchFn(scope);
        if (!self.$$areEqual(newValue, oldValue, false)) {
            changeCount++;
        }
        oldValue = newValue;
        return changeCount;
    };
    var internalListenerFn = function() {
        listenerFn(newValue, oldValue, self);
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```

$$areEqual 方法的最后一个参数是 false, 告诉我们它没有使用值比较, 在这种情况下, 我们只比较引用.

现在, 我们拥有 $watchCollection 的基本结构. 我们可以将注意力转移到对集合改变检测.