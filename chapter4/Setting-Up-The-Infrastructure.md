### 设置基础设施

让我们在 Scope 上创建一个 $watchCollection 函数存根.

该函数写法与 $watch 方法非常相似: 它需要 watch 函数(返回我们想要监控的数据)和 listener 函数(当监控的数据发生变化时, 该函数被调用)作为参数.

在它的内部, 用两个本地版本的 watch 函数和 listener 函数作为参数调用 $watch 函数.

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
    var internalWatchFn = function(scope) {
    };
    var internalListenerFn = function() {
    };
    return this.$watch(internalWatchFn, internalListenerFn);
};
```

你或许会想 $watch 函数返回一个具有删除该监听器的函数. 通过返回的函数, $watchCollection 也具备这样的可能.

让我们为测试设置一个 describe 块, 就和之前的章节做的一样.

```js
describe('$watchCollection', function() {
    var scope;
    beforeEach(function() {
        scope = new Scope();
    });
});
```
