> axios 版本号 0.21.1

## 目录结构

```txt
   lib                                # 源码目录
    │  axios.js                       # 入口，用作创建构造函数
    │  defaults.js                    # 默认配置
    │  utils.js                       # 工具函数
    │
    ├─core
    │      Axios.js                   # Axios构造函数
    │      buildFullPath.js           # 组合url
    │      createError.js             # 创建异常
    │      dispatchRequest.js         # 派发请求
    │      enhanceError.js            # 处理异常
    │      InterceptorManager.js      # 拦截器构造函数
    │      mergeConfig.js             # 合并配置参数
    │      settle.js                  # 根据http响应状态，改变Promise的状态
    │      transformData.js           # 转换数据格式
    │
    ├─adapters                        # 请求适配器相关
    ├─cancel                          # 取消请求相关
    └─helpers                         # 辅助方法
```

## 入口

> lib/axios.js
> 设计模式：工厂模式

1. 导入一些工具函数 utils、Axios 构造函数、默认配置 defaults 等
2. 创建 Axios 实例，拷贝 Axios.prototype.request 、 Axios.prototype 等
3. 实例 API 的实现，暴露 Axios 类、all/spread、isAxiosError、create 工厂函数等

```js
'use strict';

var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var mergeConfig = require('./core/mergeConfig');
var defaults = require('./defaults');

/**
 * 创建一个Axios的实例
 *
 * @param {Object} defaultConfig 实例的默认配置
 * @return {Axios} 一个新的Axios实例
 */
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);

  // 实现原理：使axios()等同于Axios.prototype.request()
  // bind分析见下文
  var instance = bind(Axios.prototype.request, context);

  // 复制Axios.prototype上的属性和方法
  // 实现原理：使axios可以使用Axios.prototype上属性及方法
  // extend分析见下文
  utils.extend(instance, Axios.prototype, context);

  // 复制上下文到实例
  // 实现原理：使默认配置axios.defaults等同于new Axios().defaults和拦截器axios.interceptors等同于 new Axios().interceptors
  utils.extend(instance, context);

  // 最后返回实例对象
  return instance;
}

// 创建要导出的默认实例
var axios = createInstance(defaults);

// 暴露Axios类，允许类的继承。
axios.Axios = Axios;

// 创建新实例的工厂
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// 暴露 Cancel & CancelToken
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');

// 暴露 all/spread
axios.all = function all(promises) {
  return Promise.all(promises);
};
axios.spread = require('./helpers/spread');

// 暴露 isAxiosError
axios.isAxiosError = require('./helpers/isAxiosError');

module.exports = axios;

// 允许在TypeScript中使用默认的导入语法。
module.exports.default = axios;
```

**bind**

> lib/helpers/bind.js

```js
'use strict';

module.exports = function bind(fn, thisArg) {
  // 返回指定this的函数，其原理和Function.protype.bind一致，区别在于此时无需兼容构造函数的实例之情况
  // 源码优化点：apply方法支持传入类数组arguments，无需遍历构建数组
  return function wrap() {
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```

**utils 中的 extend**

> lib/utils.js

```js
/**
 * 拷贝属性
 *
 * @param {Object} a 被扩展的对象
 * @param {Object} b 源对象
 * @param {Object} thisArg 自定义this
 * @return {Object} 拓展后的对象
 */
function extend(a, b, thisArg) {
  forEach(b, function assignValue(val, key) {
    if (thisArg && typeof val === 'function') {
      // 拷贝方法（函数指定extend传入的this）
      a[key] = bind(val, thisArg);
    } else {
      // 拷贝属性值
      a[key] = val;
    }
  });
  return a;
}
```

**utils 中的 forEach**

> lib/utils.js
> 设计模式：迭代器模式

```js
/**
 * 迭代一个数组或一个对象，依次调用传入的回调函数。
 *
 * @param {Object|Array} obj 迭代的对象
 * @param {Function} fn 回调函数
 */
function forEach(obj, fn) {
  // 异常参数处理
  if (obj === null || typeof obj === 'undefined') {
    return;
  }

  // obj类型在不为object时默认转换为数组
  if (typeof obj !== 'object') {
    obj = [obj];
  }

  if (isArray(obj)) {
    // 遍历索引
    for (var i = 0, l = obj.length; i < l; i++) {
      fn.call(null, obj[i], i, obj);
    }
  } else {
    // 遍历键
    for (var key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)) {
        fn.call(null, obj[key], key, obj);
      }
    }
  }
}
```

## Axios 构造函数

> lib/core/Axios.js

**初始化**

```js
/**
 * 创建一个新的Axios实例
 *
 * @param {Object} instanceConfig 实例的默认配置
 */
function Axios(instanceConfig) {
  // 默认配置
  this.defaults = instanceConfig;
  // 拦截器
  this.interceptors = {
    // 请求
    request: new InterceptorManager(),
    // 响应
    response: new InterceptorManager()
  };
}
```

**Axios.prototype.request**

1. 兼容`axios('example/url,config)`语法调用
2. 合并配置
3. 设置请求方法（默认是 get）
4. 创建请求拦截器、dispatchRequest、响应拦截器组成的 Promise 链

```js
/**
 * 发出请求
 *
 * @param {Object} config 这个请求的具体配置(内部与this.defaults合并)
 */
Axios.prototype.request = function request(config) {
  // 实现原理：使之支持axios('example/url'[, config])使用
  if (typeof config === 'string') {
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }

  // 将默认配置和请求配置进行合并
  config = mergeConfig(this.defaults, config);

  // 设置 config.method
  if (config.method) {
    config.method = config.method.toLowerCase();
  } else if (this.defaults.method) {
    config.method = this.defaults.method.toLowerCase();
  } else {
    config.method = 'get';
  }

  // 过滤跳过的拦截器
  var requestInterceptorChain = [];
  var synchronousRequestInterceptors = true;
  //   遍历用户设置的请求拦截器 放到 requestInterceptorChain 数组的头部
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    if (
      typeof interceptor.runWhen === 'function' &&
      interceptor.runWhen(config) === false
    ) {
      return;
    }

    synchronousRequestInterceptors =
      synchronousRequestInterceptors && interceptor.synchronous;

    requestInterceptorChain.unshift(
      interceptor.fulfilled,
      interceptor.rejected
    );
  });

  var responseInterceptorChain = [];
  // 遍历用户设置的响应拦截器 放到 responseInterceptorChain 数组的尾部
  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });

  var promise;

  if (!synchronousRequestInterceptors) {
    // 针对synchronousRequestInterceptors为false情况

    // 按序组合拦截器和dispatchRequest（数组奇数放置成功拦截器，偶数元素放置失败拦截器）
    var chain = [dispatchRequest, undefined];
    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain.concat(responseInterceptorChain);

    promise = Promise.resolve(config);
    // 遍历数组，直到遍历 length 为 0
    while (chain.length) {
      // 两两对应执行
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }
  // 针对synchronousRequestInterceptors为true情况

  var newConfig = config;

  // 请求拦截处理
  // 遍历数组，直到遍历 length 为 0
  while (requestInterceptorChain.length) {
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }

  // 派发请求
  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }
  // 响应拦截处理
  // 遍历数组，直到遍历 length 为 0
  while (responseInterceptorChain.length) {
    // 两两对应执行
    promise = promise.then(
      responseInterceptorChain.shift(),
      responseInterceptorChain.shift()
    );
  }

  return promise;
};
```

**请求方法别名**

```js
utils.forEach(
  ['delete', 'get', 'head', 'options'],
  function forEachMethodNoData(method) {
    Axios.prototype[method] = function (url, config) {
      // 实现原理：使得axios.get()等同于Axios.prototype.request
      return this.request(
        mergeConfig(config || {}, {
          method: method,
          url: url,
          data: (config || {}).data
        })
      );
    };
  }
);

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function (url, data, config) {
    return this.request(
      mergeConfig(config || {}, {
        method: method,
        url: url,
        data: data
      })
    );
  };
});
```

## InterceptorManager 构造函数

> lib/core/InterceptorManager.js

**初始化**

```js
function InterceptorManager() {
  // 执行栈
  this.handlers = [];
}
```

**InterceptorManager.prototype.use**

```js
/**
 * 添加拦截器
 *
 * @param {Function} fulfilled 处理成功的函数
 * @param {Function} rejected 处理失败的函数
 *
 * @return {Number} 拦截器的ID
 */
InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {
  // 将新的拦截器添加至执行栈的尾部
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected,
    // 此处不解
    synchronous: options ? options.synchronous : false,
    runWhen: options ? options.runWhen : null
  });
  // 返回拦截器的索引
  return this.handlers.length - 1;
};
```

**InterceptorManager.prototype.eject**

```js
/**
 * 移除拦截器
 *
 * @param {Number} id 拦截器的ID
 */
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    // 置空操作
    this.handlers[id] = null;
  }
};
```

**InterceptorManager.prototype.forEach**

```js
/**
 * 遍历所有拦截器
 *
 * @param {Function} fn 遍历时调用的函数
 */
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    // 内部忽略null的情况
    if (h !== null) {
      fn(h);
    }
  });
};
```

## dispatchRequest 派发请求

> lib/core/dispatchRequest.js

1. 处理已经取消的请求
2. 确保 config.header 存在
3. 转换请求的数据格式
4. 扁平化 config.header
5. 删除 headers 下的部分请求方法参数

```js
'use strict';

var utils = require('./../utils');
var transformData = require('./transformData');
var isCancel = require('../cancel/isCancel');
var defaults = require('../defaults');

/**
 * 如果请求取消，则抛出一个`Cancel`
 */
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}

/**
 * 使用配置的适配器向服务器发送请求
 *
 * @param {object} config 请求的配置
 * @returns {Promise} 用于实现的Promise
 */
module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  // 确保headers存在
  config.headers = config.headers || {};

  // 转换请求数据
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );

  // 扁平化headers
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers
  );

  // 删除headers下的部分请求方法参数
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  // 检测环境对象
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(
    function onAdapterResolution(response) {
      // 处理取消请求
      throwIfCancellationRequested(config);

      // 转换响应数据
      response.data = transformData(
        response.data,
        response.headers,
        config.transformResponse
      );

      return response;
    },
    function onAdapterRejection(reason) {
      if (!isCancel(reason)) {
        // 处理取消请求
        throwIfCancellationRequested(config);

        // 转换响应数据
        if (reason && reason.response) {
          reason.response.data = transformData(
            reason.response.data,
            reason.response.headers,
            config.transformResponse
          );
        }
      }

      return Promise.reject(reason);
    }
  );
};
```

**transformData**

> lib/core/transformData.js

```js
/**
 * 转换请求或响应的数据
 *
 * @param {Object|String} data 需要转换的数据
 * @param {Array} headers 请求或响应的headers
 * @param {Array|Function} fns 函数/函数数组
 * @returns {*} 转换后的数据
 */
module.exports = function transformData(data, headers, fns) {
  utils.forEach(fns, function transform(fn) {
    data = fn(data, headers);
  });

  return data;
};
```

**adapter**

> lib/defaults.js
> 设计模式：适配器模式

```js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    // 浏览器使用XHR适配器
    adapter = require('./adapters/xhr');
  } else if (
    typeof process !== 'undefined' &&
    Object.prototype.toString.call(process) === '[object process]'
  ) {
    // node使用HTTP适配器
    adapter = require('./adapters/http');
  }
  return adapter;
}
```

## CancelToken 构造函数

> lib/cancel/CancelToken.js

**初始化**

```js
/**
 * `CancelToken'是一个可用于请求取消操作的对象
 *
 * @class
 * @param {Function} executor 执行者（类似Promise实例化的传入参数）
 */
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      // 已取消
      return;
    }
    // 构造取消的描述信息
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}
```

**CancelToken.prototype.throwIfRequested**

```js
/**
 * 如果请求取消，则抛出一个`Cancel`
 */
CancelToken.prototype.throwIfRequested = function throwIfRequested() {
  if (this.reason) {
    throw this.reason;
  }
};
```

**CancelToken.source**

```js
/**
 * 返回一个包含新的`CancelToken`的对象和一个函数，当调用时：取消`CancelToken'。
 */
CancelToken.source = function source() {
  // token是个promise，通过调用cancel变更其状态
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};
```
