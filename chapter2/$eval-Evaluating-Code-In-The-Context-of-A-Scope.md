### $eval - 基于特定的Scope解析并运行表达式

在 angular 中，可以通过多种方式在特定的 Scope 中执行一段 code 。其中最简单的方式就是 $eval ，这个方法接收一个 function 作为参数，然后立刻给该函数传入这个 scope 本身来执行它。通过 $eval 调用的返回值就是该函数执行后的返回值。同时 $eval 也接受第二个可选参数并将其简单的传入该函数中。

以下几个单元测试向我们展示了如何使用 $eval ，我们可以另起一个新的 describe 块来书写这些用例。

```js
     // test/scope_spec.js
     describe('$eval', function () {
     	var scope;
     	beforeEach(function () {
     		scope = new Scope();
     	});
     	it('executes $evaled function and returns result', function () {
     		scope.aValue = 42;
     		var result = scope.$eval(function (scope) {
     			return scope.aValue;
     		});
     		expect(result).toBe(42);
     	});
     	it('passes the second $eval argument straight through', function () {
     		scope.aValue = 42;
     		var result = scope.$eval(function (scope, arg) {
     			return scope.aValue + arg;
     		}, 2);
     		expect(result).toBe(44);
     	});
     });
 ```
 
 $eval 的实现也很简单直接：
 
 ```js
 // src/scope.js
 Scope.prototype.$eval = function(expr, locals) {
 	return expr(this, locals);
 };
 ```
 
 我们为什么要多此一举的通过 $eval 来执行函数而不是直接调用它呢？有人可能会说，$eval  将函数的执行与特定的 Scope 连接起来。并且，$eval 还是 $apply 构成的一部分，后者我们会在之后的章节中遇到。
 然而，更为重要的是，$eval 最引人关注的用途在于它使我们不只是关注原生的函数，而且能够关注表达式 - expressions 这一概念。
 比如当我们使用 $watch 时，我们就可以直接传递一个字符串的表达式给 $eval，它会帮助我们在特定的Scope中解析并且执行该表达式，针对这一功能,我们会在本书的第二部分来实现。