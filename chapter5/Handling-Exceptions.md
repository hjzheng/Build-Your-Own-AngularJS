### 异常处理

还有一件事情需要我们去做. 就是处理异常的发生. 当一个 listener 做一些引起异常的事情, 此时不应该停止事件的传播. 当前的实现不但没有那样做, 而且还将所有异常抛出.

```js
it('does not stop on exceptions on '+method, function() {
    var listener1 = function(event) {
        throw 'listener1 throwing an exception';
    };
    var listener2 = jasmine.createSpy();
    scope.$on('someEvent', listener1);
    scope.$on('someEvent', listener2);
    scope[method]('someEvent');
    expect(listener2).toHaveBeenCalled();
});
```

就和 watch, $evalAsync 和 $$postDigest 函数一样, 需要用 try..catch 语句块包裹每一个 listener 的调用并处理异常. 目前仅仅在 console 中打印出来.

```js
Scope.prototype.$$freEventOnScope = function(eventName, listenerArgs) {
    var listeners = this.$$listeners[eventName] || [];
    var i = 0;
    while (i < listeners.length) {
        if (listeners[i] === null) {
            listeners.splice(i, 1);
        } else {
            try {
                listeners[i].apply(null, listenerArgs);
            } catch (e) {
                console.error(e);
            }
            i++;
        }
    }
};
```
