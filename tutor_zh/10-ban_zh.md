# 黑白名单

黑白名单是服务治理中常见的功能，通过检查请求来源 ip 是否在黑、白名单中来决定是否放行请求。有了模块功能的支持，对脚本添加新的功能更加简单。

代码在 [10-ban](https://github.com/flomesh-io/pipy/tree/main/tutorial/10-ban)，顺便贴一下 [09-configuration](https://github.com/flomesh-io/pipy/tree/main/tutorial/09-configuration) 用于对比。

## 思路

在前面几篇，我们一直在用 `__turnDown` 这个模块开放的变量来控制处理流程，利用 `link` 过滤器的[特性](06-path-rewriting_zh.md#配置)很容易接更多的检查。

```js
.link('bypass', () => __turnDown, CHECK_MODULE)

.pipeline(CHECK_MODULE)
  .use(MODULE.js, CHECK_TYPE)
```

比如：

```js
.pipeline('routing')
  //...
  .link('bypass', () => __turnDown, 'ban')
  .link('bypass', () => __turnDown, 'jwt')
  //...

.pipeline('ban')
  .use('ban.js', 'check')

.pipeline('jwt')
  .use('jwt.js', 'verify')
  
.pipeline('bypass')  
```

这种类似一种“串联”的方式，将各种检查的模块组装起来。在模块中，检查通过则放行请求，返回原始的消息；不通过，则返回一个新的包含错误信息的消息，同时将检查结果（本脚本使用 `__turnDown`）返回给下一个过滤器。

在 `ban.js` 模块中同样使用了配置与逻辑分离的方式，配置 `ban.json` 中配置了服务 `Service` 的黑白名单列表。

```json
{
  "services": {
    "service-1": {
      "white": [],
      "black": [
        "127.0.0.1",
        "::1",
        "::ffff:127.0.0.1"
      ]  
    }
  }
}
```

在逻辑脚本中引入这个配置，用服务的黑白名单列表初始化全局变量 `_services`。

```js
(config =>

pipy({
  _services: (
    Object.fromEntries(
      Object.entries(config.services).map(
        ([k, v]) => [
          k,
          {
            white: (
              v.white?.length > 0 ? (
                Object.fromEntries(
                  v.white.map(
                    ip => [ip, true]
                  )
                )
              ) : null
            ),
            black: (
              v.black?.length > 0 ? (
                Object.fromEntries(
                  v.black.map(
                    ip => [ip, true]
                  )
                )
              ) : null
            ),
          }
        ]
      )
    )
  ),

  _service: null,
})

//...

)(JSON.decode(pipy.load('ban.json')))
```

检查的核心逻辑：

```js
.import({
  __turnDown: 'proxy',
  __serviceID: 'proxy',
})

.pipeline('check')
  .handleSessionStart(
    () => (
      _service = _services[__serviceID],
      __turnDown = Boolean(
        _service && (
          _service.white ? (
            !_service.white[__inbound.remoteAddress]
          ) : (
            _service.black?.[__inbound.remoteAddress]
          )
        )
      )
    )
  )
  .link(
    'deny', () => __turnDown,
    'bypass'
  )

.pipeline('deny')
  .replaceMessage(
    new Message({ status: 403 }, 'Access denied')
  )

.pipeline('bypass')
```

这里使用了 `handleSessionStart` 过滤器。

## 过滤器

### handleSessionStart 过滤器

黑白名单的检查是检查请求来源的 ip 地址，无需 7 层 HTTP 的解帧。通过 Pipy 内置的全局变量 `__inbound` 的 `remoteAddress` 即可访问到来源 ip。

```js
.handleSessionStart(
    () => (
      _service = _services[__serviceID],
      __turnDown = Boolean(
        _service && (
          _service.white ? (
            !_service.white[__inbound.remoteAddress]
          ) : (
            _service.black?.[__inbound.remoteAddress]
          )
        )
      )
    )
  )

```

## 思考

这里我们探索了“串联”方式组合模块来实现服务治理，下一篇我们尝试下流量控制。