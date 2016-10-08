### 创建 $compile Provider

指令的编译发生是因为使用一个叫做 $compile 的函数. 它和 $rootScope, $parse, $q 和 $http 一样, 是由 injector 提供的内建的服务. 当我们使用 ng 模块创建一个 injector. 我们期望 $compile 已经有了.

```js
it('sets up $compile', function() {
    publishExternalAPI();
    var injector = createInjector(['ng']);
    expect(injector.has('$compile')).toBe(true);
});
```

$compile 在文件 compile.js 中被定义为一个 provider.

```js
function $CompileProvoder() {
    this.$get = function() {
      
    };
}

module.exports = $CompileProvoder;
```

我们在 angular_public.js 中的 ng 模块中引入 $compile, 就像引入其他 service 一样:

```js
function publishExternalAPI() {
    setupModuleLoader(window);
    var ngModule = angular.module('ng', []);
    ngModule.provider('$flter', require('./flter'));
    ngModule.provider('$parse', require('./parse'));
    ngModule.provider('$rootScope', require('./scope'));
    ngModule.provider('$q', require('./q').$QProvider);
    ngModule.provider('$$q', require('./q').$$QProvider);
    ngModule.provider('$httpBackend', require('./http_backend'));
    ngModule.provider('$http', require('./http').$HttpProvider);
    ngModule.provider('$httpParamSerializer',
    require('./http').$HttpParamSerializerProvider);
    ngModule.provider('$httpParamSerializerJQLike',
    require('./http').$HttpParamSerializerJQLikeProvider);
    ngModule.provider('$compile', require('./compile'));
}
```