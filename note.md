# 阅读源码

1. git checkout 0.24.0

2. 看 README.md，package.json

3. 从入口文件开始看

4. index.js

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);
  // 。。。
  instance.create = function create(instanceConfig) {
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };

  return instance;
}

// Create the default instance to be exported
var axios = createInstance(defaults);

// Expose Axios class to allow class inheritance
axios.Axios = Axios;

module.exports = axios;

// Axios  ==> createInstance
```

5. lib.core.Axios.js

```js
Axios.prototype.request = function request(config) {
  config = mergeConfig(this.defaults, config);

  // filter out skipped interceptors
  var requestInterceptorChain = [];
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    requestInterceptorChain.unshift(
      interceptor.fulfilled,
      interceptor.rejected
    );
  });

  var responseInterceptorChain = [];
  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });

  var newConfig = config;
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

  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }

  while (responseInterceptorChain.length) {
    promise = promise.then(
      responseInterceptorChain.shift(),
      responseInterceptorChain.shift()
    );
  }

  return promise;
};

// Provide aliases for supported request methods
utils.forEach(
  ["delete", "get", "head", "options"],
  function forEachMethodNoData(method) {
    /*eslint func-names:0*/
    Axios.prototype[method] = function (url, config) {
      return this.request(
        mergeConfig(config || {}, {
          method: method,
          url: url,
          data: (config || {}).data,
        })
      );
    };
  }
);

utils.forEach(["post", "put", "patch"], function forEachMethodWithData(method) {
  /*eslint func-names:0*/
  Axios.prototype[method] = function (url, data, config) {
    return this.request(
      mergeConfig(config || {}, {
        method: method,
        url: url,
        data: data,
      })
    );
  };
});

module.exports = Axios;

// this.request  ==> dispatchRequest
```

6. lib/core/dispatchRequest.js

1. 取消请求的处理和判断
1. 处理 参数和默认参数
1. 使用相对应的环境 adapter 发送请求(浏览器环境使用 XMLRequest 对象、Node 使用 http 对象)
1. 返回后抛出取消请求 message，根据配置 transformData 转换 响应数据

```js
/**
 * Dispatch a request to the server using the configured adapter.
 * @param {object} config The config that is to be used for the request
 * @returns {Promise} The Promise to be fulfilled
 */
module.exports = function dispatchRequest(config) {
  var adapter = config.adapter || defaults.adapter;
  return adapter(config).then(
    function onAdapterResolution(response) {
      // 省略。。。
      return response;
    },
    function onAdapterRejection(reason) {
      // 省略。。。
      return Promise.reject(reason);
    }
  );
};

// dispatchRequest  ==> adapter
```

7. lib/defaults.js

```js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== "undefined") {
    // For browsers use XHR adapter
    adapter = require("./adapters/xhr");
  } else if (
    typeof process !== "undefined" &&
    Object.prototype.toString.call(process) === "[object process]"
  ) {
    // For node use HTTP adapter
    adapter = require("./adapters/http");
  }
  return adapter;
}

// 浏览器端和Node端有不同实现
```

8. lib/adapters/xhr.js

```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var request = new XMLHttpRequest();

    request.open(
      config.method.toUpperCase(),
      buildURL(fullPath, config.params, config.paramsSerializer),
      true
    );

    // Set the request timeout in MS
    request.timeout = config.timeout;

    function onloadend() {
      if (!request) {
        return;
      }
      settle(
        function _resolve(value) {
          resolve(value);
          done();
        },
        function _reject(err) {
          reject(err);
          done();
        },
        response
      );

      // Clean up request
      request = null;
    }

    if ("onloadend" in request) {
      // Use onloadend if available
      request.onloadend = onloadend;
    } else {
      // Listen for ready state to emulate onloadend
      request.onreadystatechange = function handleLoad() {
        if (!request || request.readyState !== 4) {
          return;
        }

        // The request errored out and we didn't get a response, this will be
        // handled by onerror instead
        // With one exception: request that using file: protocol, most browsers
        // will return status as 0 even though it's a successful request
        if (
          request.status === 0 &&
          !(request.responseURL && request.responseURL.indexOf("file:") === 0)
        ) {
          return;
        }
        // readystate handler is calling before onerror or ontimeout handlers,
        // so we should call onloadend on the next 'tick'
        setTimeout(onloadend);
      };
    }

    // Handle browser request cancellation (as opposed to a manual cancellation)
    request.onabort = function handleAbort() {
      // 省略。。。
    };

    // Handle low level network errors
    request.onerror = function handleError() {
      // 省略。。。
    };

    // Handle timeout
    request.ontimeout = function handleTimeout() {
      // 省略。。。
    };

    // Send the request
    request.send(requestData);
  });
};

// var request = new XMLHttpRequest();
// request.open()
// request.onloadend / request.onreadystatechange
// request.send(requestData);

// resolve, reject交给了settle
// settle 函数根据状态码validateStatus判断resolve还是reject
```

9. 以 get 请求为例，函数执行流程为：
   Axios.prototype.get ==> Axios.prototype.request ==> dispatchRequest ==> xhrAdapter ==> settle
   我们大致清楚了一个请求发出需要经历的步骤

10. 数据转换和请求拦截又是怎么做的呢?
    transformRequest 和 transformResponse ，前者用来转换请求发送前的数据或请求头，后者用来转换收到而响应数据

11. lib/core/dispatchRequest.js

```js
module.exports = function dispatchRequest(config) {
  // Ensure headers exist
  config.headers = config.headers || {};

  // Transform request data
  config.data = transformData.call(
    config,
    config.data,
    config.headers,
    config.transformRequest
  );

  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(
    function onAdapterResolution(response) {
      // Transform response data
      response.data = transformData.call(
        config,
        response.data,
        response.headers,
        config.transformResponse
      );

      return response;
    },
    function onAdapterRejection(reason) {
      if (!isCancel(reason)) {
        // Transform response data
        if (reason && reason.response) {
          reason.response.data = transformData.call(
            config,
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

// 两种数据转换：一个是在 XHR send 之前，一个是在 Promise resolve 后。
```

12. lib/core/Axios.js

```js
Axios.prototype.request = function request(config) {
  config = mergeConfig(this.defaults, config);

  // filter out skipped interceptors
  var requestInterceptorChain = [];
  var synchronousRequestInterceptors = true;
  this.interceptors.request.forEach(function unshiftRequestInterceptors(
    interceptor
  ) {
    synchronousRequestInterceptors =
      synchronousRequestInterceptors && interceptor.synchronous;
    requestInterceptorChain.unshift(
      interceptor.fulfilled,
      interceptor.rejected
    );
  });

  var responseInterceptorChain = [];
  this.interceptors.response.forEach(function pushResponseInterceptors(
    interceptor
  ) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });

  var promise;

  if (!synchronousRequestInterceptors) {
    var chain = [dispatchRequest, undefined];

    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain = chain.concat(responseInterceptorChain);

    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }

  // 请求拦截
  var newConfig = config;
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

  // dispatchRequest 内部有数据转换
  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }

  // 响应拦截
  while (responseInterceptorChain.length) {
    promise = promise.then(
      responseInterceptorChain.shift(),
      responseInterceptorChain.shift()
    );
  }

  return promise;
};

// requestInterceptorChain 和 responseInterceptorChain
// 数组存放着fulfilled和rejected函数
// dispatchRequest负责数据转换
```

![Promise链](https://pic4.zhimg.com/80/v2-6075b1ceb55311820cac33da4334a92f_720w.jpg)

13. 如何取消请求？

```js
// lib/cancel/CancelToken.js
function CancelToken(executor) {
  if (typeof executor !== "function") {
    throw new TypeError("executor must be a function.");
  }

  var resolvePromise;

  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;

  this.promise.then(function (cancel) {
    if (!token._listeners) return;

    var i;
    var l = token._listeners.length;

    for (i = 0; i < l; i++) {
      token._listeners[i](cancel);
    }
    token._listeners = null;
  });

  // eslint-disable-next-line func-names
  this.promise.then = function (onfulfilled) {
    var _resolve;
    // eslint-disable-next-line func-names
    var promise = new Promise(function (resolve) {
      token.subscribe(resolve);
      _resolve = resolve;
    }).then(onfulfilled);

    promise.cancel = function reject() {
      token.unsubscribe(_resolve);
    };

    return promise;
  };

  executor(function cancel(message) {
    if (token.reason) {
      // Cancellation has already been requested
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

// 取消请求有两种情况：
//  1. 发出请求之前，此时请求报错不会被发出
//  2. 响应到达客户端之前取消了。这个请求会达到.catch里
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }

  // 此处会Cancel，在请求发出之前
  if (config.signal && config.signal.aborted) {
    throw new Cancel("canceled");
  }
}

module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(
    function onAdapterResolution(response) {
      throwIfCancellationRequested(config);

      // Transform response data
      response.data = transformData.call(
        config,
        response.data,
        response.headers,
        config.transformResponse
      );

      return response;
    },
    function onAdapterRejection(reason) {
      if (!isCancel(reason)) {
        throwIfCancellationRequested(config);

        // Transform response data
        if (reason && reason.response) {
          reason.response.data = transformData.call(
            config,
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

module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var requestHeaders = config.headers;
    var responseType = config.responseType;
    var onCanceled;

    // done函数在多处都有调用
    function done() {
      if (config.cancelToken) {
        config.cancelToken.unsubscribe(onCanceled);
      }

      if (config.signal) {
        config.signal.removeEventListener("abort", onCanceled);
      }
    }

    var request = new XMLHttpRequest();

    function onloadend() {
      if (!request) {
        return;
      }
      settle(
        function _resolve(value) {
          resolve(value);
          done();
        },
        function _reject(err) {
          reject(err);
          done();
        },
        response
      );

      // Clean up request
      request = null;
    }

    // onloadend/onreadystatechange

    // Handle browser request cancellation (as opposed to a manual cancellation)
    request.onabort = function handleAbort() {
      if (!request) {
        return;
      }

      reject(createError("Request aborted", config, "ECONNABORTED", request));

      // Clean up request
      request = null;
    };

    if (config.cancelToken || config.signal) {
      // Handle cancellation
      // eslint-disable-next-line func-names
      onCanceled = function (cancel) {
        if (!request) {
          return;
        }
        reject(
          !cancel || (cancel && cancel.type) ? new Cancel("canceled") : cancel
        );
        request.abort();
        request = null;
      };

      config.cancelToken && config.cancelToken.subscribe(onCanceled);
      if (config.signal) {
        config.signal.aborted
          ? onCanceled()
          : config.signal.addEventListener("abort", onCanceled);
      }
    }

    // Send the request
    request.send(requestData);
  });
};
```

14. 客户端支持防范 CSRF

```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    // 。。。
    var request = new XMLHttpRequest();

    function onloadend() {
      if (!request) {
        return;
      }

      settle(
        function _resolve(value) {
          resolve(value);
          done();
        },
        function _reject(err) {
          reject(err);
          done();
        },
        response
      );

      // Clean up request
      request = null;
    }

    // onloadend/onreadystatechange

    // Add xsrf header
    // This is only done if running in a standard browser environment.
    // Specifically not if we're in a web worker, or react-native.
    // web worker 和 react-native 里面是无效的
    if (utils.isStandardBrowserEnv()) {
      // Add xsrf header
      var xsrfValue =
        (config.withCredentials || isURLSameOrigin(fullPath)) &&
        config.xsrfCookieName
          ? cookies.read(config.xsrfCookieName)
          : undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }

    // Add withCredentials to request if needed
    if (!utils.isUndefined(config.withCredentials)) {
      request.withCredentials = !!config.withCredentials;
    }

    // Send the request
    request.send(requestData);
  });
};
```
