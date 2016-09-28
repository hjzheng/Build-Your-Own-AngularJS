### 处理重复代码

我们定义了两个几乎相同的测试和两个相同的函数. 很明显两种事件传播机制会有许多相似. 让我们在进行下一步之前处理下重复代码:

对于事件方法本身, 我们可以抽取共同的行为作为一个函数在 $emit 和 $broadcast 中使用, 让我们称呼它为 $$fireEventOnScope:

```js
Scope.prototype.$emit = function(eventName) {
    this.$$fireEventOnScope(eventName);
};

Scope.prototype.$broadcast = function(eventName) {
    this.$$fireEventOnScope(eventName);
};

Scope.prototype.$$fireEventOnScope = function(eventName) {
    var listeners = this.$$listeners[eventName] || [];
    _.forEach(listeners, function(listener) {
        listener();
    });
};
```

这样好多了, 但是我们可以更进一步, 消除测试中的重复. 我们可以用一个循环包装公共的功能分别为 $emit 和 $broadcast 运行一次. 在循环体内, 我们可以动态的查找正确的函数, 代替之前添加的测试中的函数:

```js
_.forEach(['$emit', '$broadcast'], function(method) {
  it('calls listeners registered for matching events on '+method, function() {
    var listener1 = jasmine.createSpy();
    var listener2 = jasmine.createSpy();
    scope.$on('someEvent', listener1);
    scope.$on('someOtherEvent', listener2);
    scope[method]('someEvent');
    expect(listener1).toHaveBeenCalled();
    expect(listener2).not.toHaveBeenCalled();
  });
});
```

因为 Jasmine 的 describe 块也是函数, 我们可以在它们中运行任意代码. 我们的循环有效的为两个 describe 块定义了测试用例.
