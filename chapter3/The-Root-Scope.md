### Root Scope

目前, 我们已经使用过通过 Scope 构造函数创建的 scope 对象:

```js
var scope = new Scope();
```

Root Scope 也是像上面一样的方式创建的, 之所以叫 Root Scope, 是因为它没有父亲, 通常情况是一个 Child Scope 树的根.

在真实 Angular 应用中, 你不可能用这种方式创建一个 Root Scope, 因为已经存在一个 Root Scope, 其他所有的 Scope 都是它的子孙, 不论是被控制器还是指令创建的.