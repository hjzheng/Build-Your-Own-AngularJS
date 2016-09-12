### 创建一个子 Scope

尽管你可以创建很多的 Root Scope, 但是通常情况是在一个已存在的 Scope 上创建一个子的 Scope. 只需要在已存在的 Scope 上调用 $new 方法就行.

让我们使用测试驱动的方式来实现 $new 方法, 在开始前, 首先在 test/scope_spec.js 上添加一个描述让所有的测试去继承, 该测试文件应该有像下面的结构:

```js
describe('Scope', function() {
  describe('digest', function() {
    // Tests from the previous chapter...
  });
  describe('$watchGroup', function() {
    // Tests from the previous chapter...
  });
  describe('inheritance', function() {
    // Tests for this chapter
  });
});
```

第一件事情就是关于子 scope 可以共享父 scope 的属性.

```js
it('inherits the parent\'s properties', function() {
  var parent = new Scope();
  parent.aValue = [1, 2, 3];
  var child = parent.$new();
  expect(child.aValue).toEqual([1, 2, 3]);
});
```
同样的一个定义在子 scope 上的属性不应该存在于父 scope 上.

```js
it('does not cause a parent to inherit its properties', function() {
  var parent = new Scope();
  var child = parent.$new();
  child.aValue = [1, 2, 3];
  expect(parent.aValue).toBeUnde ned();
});
```
共享属性和属性什么时候定义没有关系, 当一个属性定义在父 scope 上, 它所有的子 scope 都可以访问这个属性.

```js
it('inherits the parents properties whenever they are de ned', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = [1, 2, 3];
  expect(child.aValue).toEqual([1, 2, 3]);
});
```
你可以在子 scope 上操作父 scope 上的属性, 实际上它们指向同样的值.

```js
it('can manipulate a parent scopes property', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = [1, 2, 3];
  child.aValue.push(4);
  expect(child.aValue).toEqual([1, 2, 3, 4]);
  expect(parent.aValue).toEqual([1, 2, 3, 4]);
});
```

你可以在子 scope 上监控父 scope 上的属性

```js
it('can watch a property in the parent', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.aValue = [1, 2, 3];
  child.counter = 0;
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    },
    true
  );
  child.$digest();
  expect(child.counter).toBe(1);
  parent.aValue.push(4);
  child.$digest();
  expect(child.counter).toBe(2);
});
```

最后, scope 层级关系可以是任意深度.

```js
it('can be nested at any depth', function() {
  var a   = new Scope();
  var aa  = a.$new();
  var aaa = aa.$new();
  var aab = aa.$new();
  var ab  = a.$new();
  var abb = ab.$new();
  a.value = 1;

  expect(aa.value).toBe(1);
  expect(aaa.value).toBe(1);
  expect(aab.value).toBe(1);
  expect(ab.value).toBe(1);
  expect(abb.value).toBe(1);

  ab.anotherValue = 2;
  expect(abb.anotherValue).toBe(2);
  expect(aa.anotherValue).toBeUnde ned();
  expect(aaa.anotherValue).toBeUnde ned();
});
```

目前这种实现是非常直接, 我们只需要深入理解 JavaScript 对象继承, 因为 Angular 刻意模仿 JavaScript 本身如何工作. 本质上来说, 当你创建一个子的 scope 它的父 scope 将会成为它的原型.

```js
Scope.prototype.$new = function() {
  var ChildScope = function() { };
  ChildScope.prototype = this;
  var child = new ChildScope();
  return child;
};
```

在这个函数中, 首先, 我们为子 scope 创建一个构造函数, 将它存在一个局部变量 ChildScope, 然后将 Scope 作为 ChildScope 的原型, 最终我们创建一个 ChildScope 构造器 并返回它.
