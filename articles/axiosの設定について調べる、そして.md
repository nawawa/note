## インストール

```bash
yarn add @nuxtjs/axios
```

そして、nuxt.config.jsのmodulesセクションに追加してください。

```js
export default {
  modules: ['@nuxtjs/axios']
}
```

以上です。これで、Nuxtアプリで`$axios`（グローバル変数）が使えるようになりました。 ✨

## 設定

nuxt.config.js に axios オブジェクトを追加して、すべてのリクエストに適用されるグローバルなオプションを設定します。

```js
export default {
  modules: [
    '@nuxtjs/axios',
  ],

  axios: {
    // proxy: true
  }
}
```

---

## オプション

nuxt.config.js の axios プロパティを使って、異なるオプションを渡すことができます。

```javascript
export default {
  axios: {
    // Axios options here
  }
}
```

## Runtime Config

実運用で環境変数を使用する場合は、実行時設定の使用が必須です。
そうしないと、ビルド時に値がハードコーディングされ、変更されなくなります。

```
ハードコーディング：ソース中に固定の値が埋め込まれること

そういう値は環境変数なり定数なりから実行時に呼び出されるのが望ましい(ソフトコーディング)
```

サポートされているオプション

- `baseURL`
- `browserBaseURL`

### nuxt.config.js

```js
export default {
  modules: [
    '@nuxtjs/axios'
  ],

  axios: {
    baseURL: 'http://localhost:4000', // ランタイムコンフィグが提供されない場合のフォールバックとして使用されます。
  },

  publicRuntimeConfig: {
    axios: {
      browserBaseURL: process.env.BROWSER_BASE_URL
    }
  },

  privateRuntimeConfig: {
    axios: {
      baseURL: process.env.BASE_URL
    }
  },
}
```

## prefix, host and port

これらのオプションは、`baseURL` と `browserBaseURL` のデフォルト値として使用されます。

これらは環境変数 API_PREFIX, API_HOST (または HOST), API_PORT (または PORT) でカスタマイズすることができます。

`prefix` のデフォルト値は / です。

## baseURL

- Default: `http://[HOST]:[PORT][PREFIX]`

サーバーサイドのリクエストを行う際に使用されるベース URL を定義する。

環境変数 `API_URL` は、 `baseURL` を上書きするために使用することができる。

**警告: ** `baseURL` と `proxy` は同時に使用することができないため、 `proxy` オプションを使用している場合は、 `baseURL` の代わりに `prefix` を定義する必要がある。

## browserBaseURL

- Default: `baseURL`

クライアントサイドのリクエストを行う際に使用するベース URL を定義し、その先頭に追加します。

環境変数 `API_URL_BROWSER` は、 `browserBaseURL` を上書きするために使用することができます。

**警告:** `proxy` オプションが有効な場合、 `browserBaseURL` のデフォルトは `baseURL` の代わりに `prefix` になります。

## https

- Default: `false`

true` に設定すると、 `baseURL` と `browserBaseURL` の両方にある `http://` は `https://` に変更されます。

## proxy

- Default: `false`

Axios と Proxy モジュールを簡単に統合することができます。これは、CORS と生産/配備の問題を防ぐために強く推奨されます。

```js
{
  modules: [
    '@nuxtjs/axios'
  ],

  axios: {
    proxy: true // デフォルトのオプションを持つオブジェクトも可能です。
  },

  proxy: {
    '/api/': 'http://api.example.com',
    '/api2/': 'http://api.another-website.com'
  }
}
```

Note: 手動で @nuxtjs/proxy モジュールを登録する必要はありませんが、依存関係にあるモジュールである必要があります。

Note: proxy モジュールでは、API エンドポイントへのすべてのリクエストに /api/ が追加されます。これを削除したい場合は、pathRewrite オプションを使用します。

```js
proxy: {
  '/api/': { target: 'http://api.example.com', pathRewrite: {'^/api/': ''} }
}
```

## credentials

- Default: `false`

バックエンドに認証ヘッダを渡す必要があるリクエストを baseURL に発行する際に、withCredentials axios 設定を自動的に設定するインターセプターを追加します。

```js
'use strict';

var utils = require('./../utils');
var settle = require('./../core/settle');
var cookies = require('./../helpers/cookies');
var buildURL = require('./../helpers/buildURL');
var buildFullPath = require('../core/buildFullPath');
var parseHeaders = require('./../helpers/parseHeaders');
var isURLSameOrigin = require('./../helpers/isURLSameOrigin');
var transitionalDefaults = require('../defaults/transitional');
var AxiosError = require('../core/AxiosError');
var CanceledError = require('../cancel/CanceledError');
var parseProtocol = require('../helpers/parseProtocol');

module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    var requestData = config.data;
    var requestHeaders = config.headers;
    var responseType = config.responseType;
    var onCanceled;
    function done() {
      if (config.cancelToken) {
        config.cancelToken.unsubscribe(onCanceled);
      }

      if (config.signal) {
        config.signal.removeEventListener('abort', onCanceled);
      }
    }

    if (utils.isFormData(requestData) && utils.isStandardBrowserEnv()) {
      delete requestHeaders['Content-Type']; // Let the browser set it
    }

    var request = new XMLHttpRequest();

    // HTTP basic authentication
    if (config.auth) {
      var username = config.auth.username || '';
      var password = config.auth.password ? unescape(encodeURIComponent(config.auth.password)) : '';
      requestHeaders.Authorization = 'Basic ' + btoa(username + ':' + password);
    }

    var fullPath = buildFullPath(config.baseURL, config.url);

    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);

    // Set the request timeout in MS
    request.timeout = config.timeout;

    function onloadend() {
      if (!request) {
        return;
      }
      // Prepare the response
      var responseHeaders = 'getAllResponseHeaders' in request ? parseHeaders(request.getAllResponseHeaders()) : null;
      var responseData = !responseType || responseType === 'text' ||  responseType === 'json' ?
        request.responseText : request.response;
      var response = {
        data: responseData,
        status: request.status,
        statusText: request.statusText,
        headers: responseHeaders,
        config: config,
        request: request
      };

      settle(function _resolve(value) {
        resolve(value);
        done();
      }, function _reject(err) {
        reject(err);
        done();
      }, response);

      // Clean up request
      request = null;
    }

    if ('onloadend' in request) {
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
        if (request.status === 0 && !(request.responseURL && request.responseURL.indexOf('file:') === 0)) {
          return;
        }
        // readystate handler is calling before onerror or ontimeout handlers,
        // so we should call onloadend on the next 'tick'
        setTimeout(onloadend);
      };
    }

    // Handle browser request cancellation (as opposed to a manual cancellation)
    request.onabort = function handleAbort() {
      if (!request) {
        return;
      }

      reject(new AxiosError('Request aborted', AxiosError.ECONNABORTED, config, request));

      // Clean up request
      request = null;
    };

    // Handle low level network errors
    request.onerror = function handleError() {
      // Real errors are hidden from us by the browser
      // onerror should only fire if it's a network error
      reject(new AxiosError('Network Error', AxiosError.ERR_NETWORK, config, request, request));

      // Clean up request
      request = null;
    };

    // Handle timeout
    request.ontimeout = function handleTimeout() {
      var timeoutErrorMessage = config.timeout ? 'timeout of ' + config.timeout + 'ms exceeded' : 'timeout exceeded';
      var transitional = config.transitional || transitionalDefaults;
      if (config.timeoutErrorMessage) {
        timeoutErrorMessage = config.timeoutErrorMessage;
      }
      reject(new AxiosError(
        timeoutErrorMessage,
        transitional.clarifyTimeoutError ? AxiosError.ETIMEDOUT : AxiosError.ECONNABORTED,
        config,
        request));

      // Clean up request
      request = null;
    };

    // Add xsrf header
    // This is only done if running in a standard browser environment.
    // Specifically not if we're in a web worker, or react-native.
    if (utils.isStandardBrowserEnv()) {
      // Add xsrf header
      var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ?
        cookies.read(config.xsrfCookieName) :
        undefined;

      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }

    // Add headers to the request
    if ('setRequestHeader' in request) {
      utils.forEach(requestHeaders, function setRequestHeader(val, key) {
        if (typeof requestData === 'undefined' && key.toLowerCase() === 'content-type') {
          // Remove Content-Type if data is undefined
          delete requestHeaders[key];
        } else {
          // Otherwise add header to the request
          request.setRequestHeader(key, val);
        }
      });
    }

    // Add withCredentials to request if needed
    if (!utils.isUndefined(config.withCredentials)) {
      request.withCredentials = !!config.withCredentials;
    }

    // Add responseType to request if needed
    if (responseType && responseType !== 'json') {
      request.responseType = config.responseType;
    }

    // Handle progress if needed
    if (typeof config.onDownloadProgress === 'function') {
      request.addEventListener('progress', config.onDownloadProgress);
    }

    // Not all browsers support upload events
    if (typeof config.onUploadProgress === 'function' && request.upload) {
      request.upload.addEventListener('progress', config.onUploadProgress);
    }

    if (config.cancelToken || config.signal) {
      // Handle cancellation
      // eslint-disable-next-line func-names
      onCanceled = function(cancel) {
        if (!request) {
          return;
        }
        reject(!cancel || (cancel && cancel.type) ? new CanceledError() : cancel);
        request.abort();
        request = null;
      };

      config.cancelToken && config.cancelToken.subscribe(onCanceled);
      if (config.signal) {
        config.signal.aborted ? onCanceled() : config.signal.addEventListener('abort', onCanceled);
      }
    }

    if (!requestData) {
      requestData = null;
    }

    var protocol = parseProtocol(fullPath);

    if (protocol && [ 'http', 'https', 'file' ].indexOf(protocol) === -1) {
      reject(new AxiosError('Unsupported protocol ' + protocol + ':', AxiosError.ERR_BAD_REQUEST, config));
      return;
    }


    // Send the request
    request.send(requestData);
  });
};
```

