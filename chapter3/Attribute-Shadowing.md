### 属性遮蔽

Scope 继承的一个方面, 属性遮蔽, 常常绊倒 Angular 新手. 然而这却是使用 JavaScript 原型链的直接结果, 它非常值得讨论.

从现有的测试用例中可以看的非常清楚, 当你从 Scope 上读一个属性, 如果属性不在当前 Scope, 它会通过原型链往上查找, 从父 Scope 上查找, 直到找到为止.

还有, 当你在一个 Scope 上添加一个属性, 这个属性在该 Scope 和它的子 Scope 上都是可用的, 它的父 Scope 上不可用.

属性遮蔽的关键实现是可以在子 Scope 上使用相同的名称的属性:

```js
it('shadows a parents property with the same name', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.name = 'Joe';
  child.name = 'Jill';
  expect(child.name).toBe('Jill');
  expect(parent.name).toBe('Joe');
});
```

当我们在子 Scope 中添加一个名称已经存在于父 Scope 中的属性时, 不会对父 Scope 产生任何改变. 实际上, 我们在 Scope 链上有两个完全不同的属性, 只是名字都叫 name 而已.
这就是所谓的遮蔽, 从子 Scope 的角度来看, 父 Scope 的 name 属性, 被子 Scope 的 name 属性遮蔽了.

那么,我们如何改变父 Scope 上属性的状态呢? 很简单, 将父 Scope 上的属性用对象包裹起来.

```js
it('does not shadow members of parent scopes attributes', function() {
  var parent = new Scope();
  var child = parent.$new();
  parent.user = {name: 'Joe'};
  child.user.name = 'Jill';
  expect(child.user.name).toBe('Jill');
  expect(parent.user.name).toBe('Jill');
});
```

这种方式工作的原因是, 我们并没有对子 Scope 上做任何赋值. 我们仅仅从父 Scope 上读取 user 对象和在对象内赋值, 两个 Scope 使用了同一个 user 对象的引用. user 对象只是一个 JavaScript 对象, 与 Scope 继承没有任何关系.