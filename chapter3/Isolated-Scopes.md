### 隔离 Scope

我们已经看到了, 父 Scope 和 子 Scope 之间的关系是非常亲密的, 当引入原型继承的时候. 无论父 Scope 上的什么属性, 子 Scope 都可以访问. 如果这些是对象或数组属性的话, 子 Scope 也可以修改它们的内容.

有时候, 我们不希望它们非常亲密, 子 Scope 只是 父 Scope 层级中的一部分, 并不能访问父 Scope 上的任何属性. 这就是隔离 Scope 的作用.

隔离 Scope 背后的想法很简单: 我们就像之前一样, 创建一个 Scope 作为父 Scope 中的一部分, 只是不再从父 Scope 做原型继承. 这相当于切断(隔离)来自父 Scope 原型链.

隔离 Scope 可以通过对 $new 方法传递一个布尔参数创建, 当这个参数为 true 时, 将会创建一个隔离 Scope, 当它为 false (或省略)时, 原型继承将会被使用.

当 Scope 是隔离的时候, 它将无法访问它父 Scope 上的属性:

```js
it('does not have access to parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  parent.aValue = 'abc';
  expect(child.aValue).toBeUnde ned();
});
```

因为无法访问父 Scope 的属性, 因此也无法 watch 这些属性.

```js
it('cannot watch parent attributes when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  parent.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  child.$digest();
  expect(child.aValueWas).toBeUndefined();
});
```

Scope 是否隔离是在 $new 中设置的. 根据一个给定的 boolean 参数, 决定创建的子 Scope 是和目前的做法一样还是使用 Scope 构造器创建独立的 Scope.

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
  } else {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    child = new ChildScope();
  }
  this.$$children.push(child);
  child.$$watchers = [];
  child.$$children = [];
  return child;
};
```

如果你使用过 Angular 指令中的隔离 Scope, 你就会知道隔离 Scope 并没有完全与它的父 Scope 隔离, 你可以显示地在父 Scope 上定义一个 Map 去从父 Scope 上获取对应的属性.

但是, 这个设计并未在 Scope 中实现, 它是指令实现的一部分.

因为破坏了原型继承链, 我们需要重新讨论 $digest, $apply, $evalAsync 和 $applyAsync 在之前章节的实现.

首先, 我们想让 $digest 运行当前 Scope 上的监听器并遍历它的所有子 Scope 上的监听器, 这个我们已经处理过了. 因为隔离 Scope 也被它们父 Scope 的 $$children 属性所包含, 所以下面的测试已经通过了:

```js
it('digests its isolated children', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  child.aValue = 'abc';
  child.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.aValueWas = newValue;
    }
  );
  parent.$digest();
  expect(child.aValueWas).toBe('abc');
});
```

这种情况下, $apply, $evalAsync 和 $applyAsync 就没有这么幸运了, 我们希望这些操作都是从 Root Scope 上开始 digest, 但是在中间层级的隔离 Scope 会破坏这种设定. 正如下面的两个失败的测试用例:

```js
it('digests from root on $apply when isolated', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  var child2 = child.$new();
  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  child2.$apply(function(scope) { });
  expect(parent.counter).toBe(1);
});

it('schedules a digest from root on $evalAsync when isolated', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);
  var child2 = child.$new();
  parent.aValue = 'abc';
  parent.counter = 0;
  parent.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );
  child2.$evalAsync(function(scope) { });
  setTimeout(function() {
    expect(parent.counter).toBe(1);
    done();
  }, 50);
});
```

两个测试用例失败的原因是, 我们依赖指向 Root Scope 的 $root 属性, 非隔离 Scope 可以从原型链上继承下来. 而隔离 Scope 却没有. 实际上, 因为我们使用 Scope 构造器创建隔离 Scope,
构造器提供了 $root 属性, 只不过每个隔离 Scope 的 $root 属性都指向它自己. 这并不是我们想要的.

简单的修复一下 $new :

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
    child.$root = this.$root;
  } else {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    child = new ChildScope();
  }
  this.$$children.push(child);
  child.$$watchers = [];
  child.$$children = [];
  return child;
};
```

在这之前, 我们已经将一切关于继承的情况都覆盖了. 还有一件事, 我们必须要在隔离 Scope 中修复, 那就是我们在 $evalAsync, $applyAsync 和 $$postDigest 函数中使用的队列:

- $$asyncQueue
- $$postDigestQueue
- $$applyAsyncQueue

对于它们, 我们并未做在子 Scope 或父 Scope 上任何额外的工作, 仅仅简单作为整个 Scope 层级上的任务队列而已.

对于非隔离 Scope: 无论在任何 Scope 上访问这个队列中一个, 我们访问的都是相同队列, 因为每个 Scope 都通过原型链继承了这些队列. 然而对这些隔离 Scope. 就如同之前的 $root 属性,
$$asyncQueue, $$applyAsyncQueue 和 $$postDigestQueue 都被 Scope 构造器创建了, 只是都指向了它们自己.

```js
it('executes $evalAsync functions on isolated scopes', function(done) {
  var parent = new Scope();
  var child = parent.$new(true);
  child.$evalAsync(function(scope) {
    scope.didEvalAsync = true;
  });
  setTimeout(function() {
    expect(child.didEvalAsync).toBe(true);
    done();
  }, 50);
});

it('executes $$postDigest functions on isolated scopes', function() {
  var parent = new Scope();
  var child = parent.$new(true);
  child.$$postDigest(function() {
    child.didPostDigest = true;
  });
  parent.$digest();
  expect(child.didPostDigest).toBe(true);
});
```

就和 $root 属性一样, 我们希望整个 Scope 层级中每个 Scope 都分享相同的队列, 不管它们是否是隔离 Scope. 如果 Scope 不是隔离的, 我们自动获得对应的队列, 如果是隔离的, 我们需要显示的赋值:

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
    child.$root = this.$root;
    child.$$asyncQueue = this.$$asyncQueue;
    child.$$postDigestQueue = this.$$postDigestQueue;
  } else {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    child = new ChildScope();
  }
  this.$$children.push(child);
  child.$$watchers = [];
  child.$$children = [];
  return child;
};
```

对于 $$applyAsyncQueue, 问题有些不太一样: 因为清理队列是被 $$applyAsyncId 属性控制的, 并且现在整个 Scope 层级中的每个 Scope 可能会有这个属性的实例, 整个 $applyAsync 的目的, 就是合并 $apply 的调用.

如果我们调用子 Scope 上 $digest 方法, 父 Scope 上 $applyAsync 注册的函数应当被清理出队列并被执行. 但是当前的实现不能是下面的测试通过:

```js
it("executes $applyAsync functions on isolated scopes", function() {
  var parent = new Scope();
  var child = parent.$new(true);
  var applied = false;
  parent.$applyAsync(function() {
    applied = true;
  });
  child.$digest();
  expect(applied).toBe(true);
});
```
首先, 我们应当在 scope 之间共享队列, 就像 $evalAsync 和 $postDigest 队列做的一样.

```js
Scope.prototype.$new = function(isolated) {
  var child;
  if (isolated) {
    child = new Scope();
    child.$root = this.$root;
    child.$$asyncQueue = this.$$asyncQueue;
    child.$$postDigestQueue = this.$$postDigestQueue;
    child.$$applyAsyncQueue = this.$$applyAsyncQueue;
  } else {
    var ChildScope = function() { };
    ChildScope.prototype = this;
    child = new ChildScope();
  }
  this.$$children.push(child);
  child.$$watchers = [];
  child.$$children = [];
  return child;
};
```

第二, 我们需要共享 $$applyAsyncId 属性, 我们不能简单的在 $new 中那样处理, 因为我们需要对它赋值(它是一个基本类型), 所以必须显示的通过 $root 访问就好了.

```js
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  this.$root.$$lastDirtyWatch = null;
  this.$beginPhase('$digest');
  if (this.$root.$$applyAsyncId) {
    clearTimeout(this.$root.$$applyAsyncId);
    this.$$flushApplyAsync();
  }
  do {
    while (this.$$asyncQueue.length) {
      try {
        var asyncTask = this.$$asyncQueue.shift();
        asyncTask.scope.$eval(asyncTask.expression);
      } catch (e) {
        console.error(e);
      }
    }
    dirty = this.$$digestOnce();
    if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
      throw '10 digest iterations reached';
    }
  } while (dirty || this.$$asyncQueue.length);

  this.$clearPhase();

  while (this.$$postDigestQueue.length) {
      try {
         this.$$postDigestQueue.shift()();
      } catch (e) {
         console.error(e);
      }
  }
};

Scope.prototype.$applyAsync = function(expr) {
  var self = this;
  self.$$applyAsyncQueue.push(function() {
    self.$eval(expr);
  });
  if (self.$root.$$applyAsyncId === null) {
    self.$root.$$applyAsyncId = setTimeout(function() {
       self.$apply(_.bind(self.$$flushApplyAsync, self));
    }, 0);
  }
};

Scope.prototype.$$ ushApplyAsync = function() {
  while (this.$$applyAsyncQueue.length) {
    try {
      this.$$applyAsyncQueue.shift()();
    } catch (e) {
      console.error(e);
    }
  }
  this.$root.$$applyAsyncId = null;
};
```