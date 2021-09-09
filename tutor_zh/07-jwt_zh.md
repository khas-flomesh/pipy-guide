# 0x07 JWT

服务系统离不开用户认证，一个复杂的业务系统通常还会用到单点登录，避免要求用户重复登录多次。通常有两种方案：

一种是将 session 数据持久化，每次都向持久层请求数据。即使加上了缓存，也存在中心化单点失败的风险。

第二种是不做持久化，将数据保存保存在客户端，每次请求发回服务端进行校验。

JWT 就是后者的一个代表。

[07-jwt/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/07-jwt/proxy.js)

```js
pipy({
  _keys: {
    'key-1': new crypto.PrivateKey(os.readFile('sample-key-rsa.pem')),
    'key-2': new crypto.PrivateKey(os.readFile('sample-key-ecdsa.pem')),
  },

  _services: {
    'service-1': {
      balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8080']),
    },
    'service-2': {
      balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082']),
      keys: { 'key-1': true, 'key-2': true },
    },
  },

  _router: new algo.URLRouter({
    '/*': {
      service: 'service-1',
    },
    '/hi/*': {
      service: 'service-2',
      rewrite: [new RegExp('^/hi'), '/hello'],
    },
  }),

  _g: {
    connectionID: 0,
  },

  _connectionPool: new algo.ResourcePool(
    () => ++_g.connectionID
  ),

  _turnDown: false,
  _route: null,
  _service: null,
  _balancerCache: null,
  _target: '',
  _targetCache: null,

  _LOG_URL: new URL('http://127.0.0.1:8123/log'),

  _request: null,
  _requestTime: 0,
  _responseTime: 0,
})

.listen(8000)
  .handleSessionStart(
    () => (
      _balancerCache = new algo.Cache(
        // k is a balancer, v is a target
        (k  ) => k.select(),
        (k,v) => k.deselect(v),
      ),
      _targetCache = new algo.Cache(
        // k is a target, v is a connection ID
        (k  ) => _connectionPool.allocate(k),
        (k,v) => _connectionPool.free(v),
      )
    )
  )
  .handleSessionEnd(
    () => (
      _balancerCache.clear(),
      _targetCache.clear()
    )
  )
  .demuxHTTP('routing')

.pipeline('routing')
  .fork('log-request')
  .handleMessageStart(
    msg => (
      _route = _router.find(
        msg.head.headers.host,
        msg.head.path,
      ),
      _route && (
        _route.rewrite && (
          msg.head.path = msg.head.path.replace(
            _route.rewrite[0],
            _route.rewrite[1],
          )
        ),
        _service = _services[_route.service]
      )
    )
  )
  .link('jwt')
  .link(
    'bypass', () => _turnDown,
    'load-balance', () => Boolean(_service),
    '404'
  )
  .fork('log-response')

.pipeline('jwt')
  .replaceMessage(
    msg => (
      ((
        header,
        jwt,
        kid,
        key,
      ) => (
        _service?.keys ? (
          header = msg.head.headers.authorization || '',
          header.startsWith('Bearer ') && (header = header.substring(7)),
          jwt = new crypto.JWT(header),
          jwt.isValid ? (
            kid = jwt.header?.kid,
            key = _keys[kid],
            key ? (
              _service.keys[kid] ? (
                jwt.verify(key) ? (
                  msg
                ) : (
                  _turnDown = true,
                  new Message({ status: 401 }, 'Invalid signature')
                )
              ) : (
                _turnDown = true,
                new Message({ status: 403 }, 'Access denied')
              )
            ) : (
              _turnDown = true,
              new Message({ status: 401 }, 'Invalid key')
            )
          ) : (
            _turnDown = true,
            new Message({ status: 401 }, 'Invalid token')
          )
        ) : msg
      ))()
    )
  )

.pipeline('load-balance')
  .handleMessageStart(
    () => _target = _balancerCache.get(_service.balancer)
  )
  .link(
    'forward', () => Boolean(_target),
    'no-target'
  )

.pipeline('forward')
  .muxHTTP(
    'connection',
    () => _targetCache.get(_target)
  )

.pipeline('connection')
  .connect(
    () => _target
  )

.pipeline('404')
  .replaceMessage(
    new Message({ status: 404 }, 'No route')
  )

.pipeline('no-target')
  .replaceMessage(
    new Message({ status: 404 }, 'No target')
  )

.pipeline('log-request')
  .handleMessageStart(
    () => _requestTime = Date.now()
  )
  .decompressHTTP()
  .handleMessage(
    '256k',
    msg => _request = msg
  )

.pipeline('log-response')
  .handleMessageStart(
    () => _responseTime = Date.now()
  )
  .decompressHTTP()
  .replaceMessage(
    '256k',
    msg => (
      new Message(
        JSON.encode({
          req: {
            ..._request.head,
            body: _request.body.toString(),
          },
          res: {
            ...msg.head,
            body: msg.body.toString(),
          },
          target: _target,
          reqTime: _requestTime,
          resTime: _responseTime,
          endTime: Date.now(),
          remoteAddr: __inbound.remoteAddress,
          remotePort: __inbound.remotePort,
          localAddr: __inbound.localAddress,
          localPort: __inbound.localPort,
        }).push('\n')
      )
    )
  )
  .merge('log-send')

.pipeline('log-send')
  .pack(
    1000,
    {
      timeout: 5,
    }
  )
  .replaceMessageStart(
    () => new MessageStart({
      method: 'POST',
      path: _LOG_URL.path,
      headers: {
        'Host': _LOG_URL.host,
        'Content-Type': 'application/json',
      }
    })
  )
  .encodeHTTPRequest()
  .connect(
    () => _LOG_URL.host,
    {
      bufferLimit: '8m',
    }
  )

.pipeline('bypass')
```

## JWT

这里简单说下 JWT 。

JWT 长成下面这样，是一个完成的字符串，没有换行。包含了由 `.` 分隔成的三部分：`Header.Payload.Signature`。每一部分都使用 `Base64URL` 计算法转成字符串。

- Header 包含了签名算法和 Token 类型
- Payload 是实际要传递的数据
- Signature 是对前面两部分的签名，防篡改

**sample-token-rsa.txt**
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0xIn0.e30.mavj3QQXhRE7-8vx4OYogfW_PrmFxOzQ9a-Fl6D-_JRhwqM3B4qkTSh7S_H-XUCvNP2TZApjlhaCiawv4Gcp8wvGlq-2gy2EHjBb2Pz_7cZ0oCqFRcd2OI19_nRcOeE6OVFNFksGgVFLGLIbZUf0Deg1v3ST1VaERK1oxuuxWTyRXFibHA6FsjNUJ30ZfBa-ewfK8SV49gOcWrT_yiL3mMfVL9_JqPHWz17jBiE0nHsUWamI7mc7hdAfMoTeD0BO-rTHVxoplgEIw6kKvqFB5MQVTL1-Uyp9bRAj2WPlObFDRrL1t5aQ-Z9DIojo9K81pPKCSIX4QEYMWjpLovoYvg
```

Pipy 内置提供了 `crypto.JWT` 用于实现 JWT 校验。

## 思路

路由决策之后，判断目标服务是否需要 JWT 校验。如果不需要直接跳过；需要的话则进行校验。校验通过后放行，失败则直接返回错误信息。

如此我们需要两个配置：
1. 标识服务是否需要校验
2. 校验记过的标识

## 配置

还记得在 [路径重写的配置](06-path-rewriting_zh.md#配置) 中抽象了一个 `Service` 类型是为了方便后面的扩展。

我们在全局变量 `_services` 的 `Service` 中增加了一个字段 `keys`。

```js
{
    //...
    _keys: {
        'key-1': new crypto.PrivateKey(os.readFile('sample-key-rsa.pem')),
        'key-2': new crypto.PrivateKey(os.readFile('sample-key-ecdsa.pem')),
    },    
    _services: {
        'service-1': {
          balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8080']),
        },
        'service-2': {
          balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082']),
          keys: { 'key-1': true, 'key-2': true },
        },
    },
    //...
}
```

`service-2` 指定了 `keys` 说明需要进行 jwt 认证，支持多种加密方式：通过 `jwtToken.head.kid` 来判断（见下文）。

通过 Pipy 内置的 `crypto.PrivateKey` 从文件读取（`os.readFile`）的内容初始化。

## 认证

在路由决策之后路由执行之前，通过 `link` 过滤器执行 pipeline `jwt`。并在路由执行中加入了对认证结果 `_turnDown` 的判读：如果认证失败，则直接返回；成功则继续执行路由。

```js
.pipeline('routing')
  .fork('log-request')
  //...
  .link('jwt')
  .link(
    'bypass', () => _turnDown,
    'load-balance', () => Boolean(_service),
    '404'
  )
  //...
```

在 pipeline `jwt` 中则是使用 `crypto.JWT` 进行 `Base64URL` 解码、验签等操作。

任何一个检查不通过，则将认证结果 `_turnDown` 设置为 `true`。

```js
.pipeline('jwt')
  .replaceMessage(
    msg => (
      ((
        header,
        jwt,
        kid,
        key,
      ) => (
        _service?.keys ? (
          header = msg.head.headers.authorization || '',
          header.startsWith('Bearer ') && (header = header.substring(7)),
          jwt = new crypto.JWT(header),
          jwt.isValid ? (
            kid = jwt.header?.kid,
            key = _keys[kid],
            key ? (
              _service.keys[kid] ? (
                jwt.verify(key) ? (
                  msg
                ) : (
                  _turnDown = true,
                  new Message({ status: 401 }, 'Invalid signature')
                )
              ) : (
                _turnDown = true,
                new Message({ status: 403 }, 'Access denied')
              )
            ) : (
              _turnDown = true,
              new Message({ status: 401 }, 'Invalid key')
            )
          ) : (
            _turnDown = true,
            new Message({ status: 401 }, 'Invalid token')
          )
        ) : msg
      ))()
    )
  )
```

## 测试

```shell
$ curl localhost:8000/hi \
-H'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0xIn0.e30.mavj3QQXhRE7-8vx4OYogfW_PrmFxOzQ9a-Fl6D-_JRhwqM3B4qkTSh7S_H-XUCvNP2TZApjlhaCiawv4Gcp8wvGlq-2gy2EHjBb2Pz_7cZ0oCqFRcd2OI19_nRcOeE6OVFNFksGgVFLGLIbZUf0Deg1v3ST1VaERK1oxuuxWTyRXFibHA6FsjNUJ30ZfBa-ewfK8SV49gOcWrT_yiL3mMfVL9_JqPHWz17jBiE0nHsUWamI7mc7hdAfMoTeD0BO-rTHVxoplgEIw6kKvqFB5MQVTL1-Uyp9bRAj2WPlObFDRrL1t5aQ-Z9DIojo9K81pPKCSIX4QEYMWjpLovoYvg'
Hi, I'm on port 8082! 
#或者 Hi, I'm on port 8081!

$ curl localhost:8000/hi
Invalid token

$ curl -i localhost:8000
HTTP/1.1 200 OK
content-length: 0
connection: keep-alive
```