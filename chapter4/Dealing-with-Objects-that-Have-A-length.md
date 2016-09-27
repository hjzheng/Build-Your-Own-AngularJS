### 处理具有 length 属性的对象

我们几乎处理完所有不同类型的集合了, 但是仍然有一种特殊的对象, 得好好考虑一下:

回顾, 我们通过检查对象是否有一个数字类型的 length 属性, 判断一个对象是否是类数组对象. 那么, 下面的对象该如何处理?

```js
{
    length: 42,
    otherKey: 'abc'
}
```

这不是一个类数组对象. 它仅仅有一个叫做 length 的属性. 既然想到这种对象有可能出现在应用中, 我们需要处理这类对象.

让我们添加一个测试用例:

```js
it('does not consider any object with a length property an array', function() {
  scope.obj = {length: 42, otherKey: 'abc'};
  scope.counter = 0;
  scope.$watchCollection(
    function(scope) { return scope.obj; },
    function(newValue, oldValue, scope) {
     scope.counter++;
    }
  );
  scope.$digest();
  scope.obj.newKey = 'def';
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

当你运行这个测试, 你会看到在对象发生变化之后, listener 函数并未被调用. 原因在于我们判断数组方法不对, 检测数组变化并不会注意新添加的属性.

修复很简单, 代替之前认为对象只要有一个数字类型的 length 属性就是类数组的写法, 让我们缩小范围: 带有数组类型 length 属性的对象, 并且这个对象有一个比这个 length 属性小于 1 的 key.

举例, 如果一个对象有一个 length 为 42 的属性, 那它也一定有叫做 41 的属性.

这种方式只针对非零长度, 所以需要对 length 为 0 的情况放宽条件.

```js
function isArrayLike(obj) {
  if (_.isNull(obj) || _.isUndefined(obj)) {
    return false;
  }
  var length = obj.length;
  return length === 0 ||
  (_.isNumber(length) && length > 0 && (length - 1) in obj);
}
```

这使我们的测试通过了, 确实对大部分对象是工作的, 但是这个检查并不能做到万无一失, 但是这已经是我们所能做到最好了.
