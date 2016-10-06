### 在销毁 Scope 时, 使 listener 失效

除了触发 $destroy 事件之外, 销毁 Scope 的另一个效果是它的事件监听器不再有效:

```js
it('no longers calls listeners after destroyed', function() {
    var listener = jasmine.createSpy();
    scope.$on('myEvent', listener);
    scope.$destroy();
    scope.$emit('myEvent');
    expect(listener).not.toHaveBeenCalled();
});
```

我们可以将 $$listeners 对象重新设置成一个空对象, 这样做实际上将丢掉所有已存在的 listener.

```js
Scope.prototype.$destroy = function() {
    this.$broadcast('$destroy');
    if (this.$parent) {
        var siblings = this.$parent.$$children;
        var indexOfThis = siblings.indexOf(this);
        if (indexOfThis >= 0) {
            siblings.splice(indexOfThis, 1);
        }
    }
    this.$$watchers = null;
    this.$$listeners = {};
};
```