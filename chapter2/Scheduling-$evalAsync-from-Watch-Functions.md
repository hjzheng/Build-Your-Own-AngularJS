### 在 Watch 函数中使用 $evalAsync
在上一小节中，当我们在 listener 函数中通过 $evalAsync 来延迟任务的执行时，该任务会在所处的脏检查循环中被执行。试想一下，当我们在 watch 函数中使用 $evalAsync 来延迟任务时，会造成什么结果？
当然，这样做本身是不应该的，因为 watch 函数本身就应该是没有副作用，没有额外的影响的。但事实上，人们仍有可能去这样做，所以，我们要确保这样做不会对脏检查过程造成大的破坏。

想象一下，我们在 watch 函数中只调用了一次 $evalAsync，那么我们目前的代码逻辑是没有问题的，也意味着下面的测试用例会通过:
```js
// test/scope_spec.js
it('executes $evalAsynced functions added by watch functions', function() {
	scope.aValue = [1, 2, 3];
	scope.asyncEvaluated = false;
	scope.$watch(
		function(scope) {
			if (!scope.asyncEvaluated) {
				scope.$evalAsync(function(scope) {
					scope.asyncEvaluated = true;
				});
			}
			return scope.aValue;
		},
		function(newValue, oldValue, scope) { }
	);
	scope.$digest();
	expect(scope.asyncEvaluated).toBe(true);
});
```
那么，问题出在哪里呢？就目前来说，只要至少存在一个“脏”的 watch，我们便会持续进行脏检查过程。上面的测试中，当我们进行对 watcher 进行第一次遍历时，watch 函数返回的 scope.aValue 检测到是“脏”的，导致我们需要进入第二次遍历，而在第二次遍历中，我们正好执行了我们在第一次中通过 $evalAsync 安排的任务。
但是，当我们调用 $evalAsync 时，整个脏检查状态是“干净”的呢，没有“脏”的 watch 呢？ 
```js
// test/scope_spec.js
it('executes $evalAsynced functions even when not dirty', function () {
	scope.aValue = [1, 2, 3];
	scope.asyncEvaluatedTimes = 0;
	scope.$watch(
		function (scope) {
			if (scope.asyncEvaluatedTimes < 2) {
				scope.$evalAsync(function (scope) {
					scope.asyncEvaluatedTimes++;
				});
			}
			return scope.aValue;
		},
		function (newValue, oldValue, scope) {
		}
	);
	scope.$digest();
	expect(scope.asyncEvaluatedTimes).toBe(2);
});
```
在这个测试用，我们在 watch 函数中调用了两次 $evalAsync，在第二次调用时，该 watch 函数检测结果是“干净”因为 scope.aValue 的值并没有改变，这也意味着第二次 $evalAsync 安排的任务不会再被执行了，因为脏检查在第二次就终止了。
相比在下一个新的脏检查中执行该任务，我们更期望该任务在这次脏检查中就被执行，这就意味着，我们需要去改变脏检查的结束条件，在结束条件中检测是否还有安排了但未执行的  async queue 任务。
```js
// src/scope.js
Scope.prototype.$digest = function() {
	var ttl = 10;
	var dirty;
	this.$$lastDirtyWatch = null;
	do {
		while (this.$$asyncQueue.length) {
			var asyncTask = this.$$asyncQueue.shift();
			asyncTask.scope.$eval(asyncTask.expression);
		}
		dirty = this.$$digestOnce();
		if (dirty && !(ttl--)) {
			throw '10 digest iterations reached';
		}
	} while (dirty || this.$$asyncQueue.length);
};
```
这使得上一个测试能够通过，但是事实上，我们又遇到了另一个问题，倘若我们在一个 watch 函数中多次调用 $evalAsync 来安排任务呢？那会发生什么？我们希望这样会触发脏检查的限制来并且停止脏检查，但目前的逻辑上并不是这样：
```js
// test/scope_spec.js
it('eventually halts $evalAsyncs added by watches', function () {
	scope.aValue = [1, 2, 3];
	scope.$watch(
		function (scope) {
			scope.$evalAsync(function (scope) {
			});
			return scope.aValue;
		}, function (newValue, oldValue, scope) {
		}
	);
	expect(function () {
		scope.$digest();
	}).toThrow();
});
```
上述的测试代码会一直执行下去，永远不会停止，因为脏检查的 while 循环永远不会停下来，现在我们需要在脏检查的终止限制中增加对 async queue 的检查来避免这种情况的发生。
```js
// src/scope.js
Scope.prototype.$digest = function() {
	var ttl = 10;
	var dirty;
	this.$$lastDirtyWatch = null;
	do {
		while (this.$$asyncQueue.length) {
			var asyncTask = this.$$asyncQueue.shift();
			asyncTask.scope.$eval(asyncTask.expression);
		}
		dirty = this.$$digestOnce();
		if ((dirty || this.$$asyncQueue.length) && !(ttl--)) {
			throw '10 digest iterations reached';
		}
	} while (dirty || this.$$asyncQueue.length);
};
```
现在我们可以保证，无论检查结果是否是“脏”以及在是否在 async queue 中还有未执行的任务，在脏检查达到我们的限制条件时，它就会终止。