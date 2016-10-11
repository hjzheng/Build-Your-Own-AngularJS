### 从函数参数中获取依赖标示

第三种也是最后一种, 或许是最最感兴趣的方式去定义一个函数的依赖, 实际上并未定义任何依赖. 当传给 injector 的函数没有带 $inject 属性并且也没有被数组包裹时, injector 将尝试从函数本身抽取依赖名称.

首先, 让我们来处理一个简单情况: 没有参数的函数.

```js
it('returns an empty array for a non-annotated 0-arg function', function() {
  var injector = createInjector([]);
  var fn = function() { };
  expect(injector.annotate(fn)).toEqual([]);
});
```

对于 annotate 方法, 当函数没有被标示依赖时, 我们返回一个空数组. 上面的测试就会通过:

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
        return fn.$inject;
    } else {
        return [];
    }
}
```

如果函数有参数, 我们需要一种方式去抽取函数的依赖名称, 为了通过下面的测试:

```js
it('returns annotations parsed from function args when not annotated', function() {
  var injector = createInjector([]);
  var fn = function(a, b) { };
  expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
```

一个小技巧是读取函数的源码然后用正在表达式提取参数声明. 在 JavaScript 中你可以通过调用函数的 toString 方法得到函数的源码:

```js
(function(a, b) { }).toString() // => "function (a, b) { }"
```

因为函数的源码包含参数列表, 所以我们可以使用下面的正则表达式抓取参数.

```js
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
```

当我们在 annotate 中使用正则表达式匹配成功时, 我们已经从捕获组中的第二项中得到了参数列表. 然后使用 `,` 去 split, 得到一个参数名称的数组. 对于没有参数的函数, 我们添加一个特殊条件去处理.

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
        return fn.$inject;
    } else if (!fn.length) {
        return [];
    } else {
        var argDeclaration = fn.toString().match(FN_ARGS);
        return argDeclaration[1].split(',');
    }
}
```

使用这个实现, 你会注意到我们测试仍然是失败的. 那是因为有一些额外的空白在第二个依赖名称里: `' b'`. 我们的正则删除参数列表开头的空白, 但是没有删除参数名称之间的空白. 为了删除这些空白, 我们需要在返回之前遍历这些参数名称.

下面的这则表达式会匹配一个字符串前后的空白, 并捕获非空白部分.

```js
var FN_ARG = /^\s*(\S+)\s*$/;
```

通过将参数名映射到这个正则表达式的第二个匹配结果，我们可以得到没有空白的参数名：

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
        return fn.$inject;
    } else if (!fn.length) {
        return [];
    } else {
        var argDeclaration = fn.toString().match(FN_ARGS);
        return _.map(argDeclaration[1].split(','), function(argName) {
            return argName.match(FN_ARG)[1];
        });
    }
}
```

简单的情况都是工作的, 但是如果在函数的声明中注释掉一些参数, 会发生什么呢?

```js
it('strips comments from argument lists when parsing', function() {
  var injector = createInjector([]);
  var fn = function(a, /*b,*/ c) { };
  expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

在抽取函数参数之前, 我们需要预先处理掉函数源码中的注释. 正则表达式如下:

```js
var STRIP_COMMENTS = /\/\*.*\*\//;
```

这个正则表达式匹配字符 `/*`, 然后是一连串的任意字符, 接着是字符 `*/`. 通过将正则表达式的匹配结果替换为空字符串，我们可以删除注释:

```js
function annotate(fn) {
    if (_.isArray(fn)) {
        return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
        return fn.$inject;
    } else if (!fn.length) {
        return [];
    } else {
        var source = fn.toString().replace(STRIP_COMMENTS, '');
        var argDeclaration = source.match(FN_ARGS);
        return _.map(argDeclaration[1].split(','), function(argName) {
            return argName.match(FN_ARG)[1];
        });
    }
}
```

当参数列表中有多个注释掉的部分时，上面的正则表达式就不对了:

```js
it('strips several comments from argument lists when parsing', function() {
    var injector = createInjector([]);
    var fn = function(a, /*b,*/ c/*, d*/) { };
    expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

该正则表达式匹配第一个 `/*` 和最后一个 `*/` 之间的一切, 因此它们之间的非注释内容也丢失了. 我们需要将开始和结束注释之间的量词转换为懒惰的，这样它会尽可能减少匹配, 我们也需要为正则表达式添加 g 标志, 确保匹配字符串中的多个注释:

```js
var STRIP_COMMENTS = /\/\*.*?\*\//g;
```

然后还有另一种类型的注释，参数列表分布在多行可能包括 `//` 的样式的注释, 注释掉一行的剩余部分:

```js
it('strips // comments from argument lists when parsing', function() {
    var injector = createInjector([]);
    var fn = function(a, //b,
                    c) { };
    expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

要删掉这些注释, 我们的 STRIP_COMMENTS 正则表达式需要匹配两类不同类型的输入: 我们早前定义的输入 和以 `//` 开头的直到行尾的注释. 我们也要为正则表达式添加 m 标志去匹配多行字符串.

```js
var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
```

这个正则表达式可以处理参数列表中所有类型注释!

The final feature we need to take care of when parsing argument names is stripping surrounding underscore characters from them. Angular lets you put an underscore character on both sides of an argument name, which it will then ignore, so that the following pattern of capturing an injected argument to a local variable with the same name is possible: