# 0x08 模块

随着功能的增加，脚本的代码量也越来越大，给升级维护增加了难度。简单的拆成几个模块行通过 `use` 过滤器引用有时不一定能解决问题，模块之前还会有数据共享（比如 [JWT]() 中使用的 `_turnDown`）。每个 `pipy({})` 中定义的全局变量只能在当前模块中访问。

为了解决这一个问题，Pipy 提供了 `import`、`export`，结合 `use` 过滤器一起解决这种问题。

[proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/08-plugins/proxy.js)

```js
pipy({
  _services: {
    'service-1': {
      balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8080']),
    },
    'service-2': {
      balancer: new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082']),
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

  _route: null,
  _balancerCache: null,
  _target: '',
  _targetCache: null,

  _LOG_URL: new URL('http://127.0.0.1:8123/log'),

  _request: null,
  _requestTime: 0,
  _responseTime: 0,
})

.export('proxy', {
  __turnDown: false,
  __serviceID: '',
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
        __serviceID = _route.service
      )
    )
  )
  .link('bypass', () => __turnDown, 'jwt')
  .link(
    'bypass', () => __turnDown,
    'load-balance', () => Boolean(__serviceID),
    '404'
  )
  .fork('log-response')

.pipeline('jwt')
  .use('jwt.js', 'verify')

.pipeline('load-balance')
  .handleMessageStart(
    () => _target = _balancerCache.get(_services[__serviceID].balancer)
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

[08-plugins/jwt.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/08-plugins/jwt.js)

```js
pipy({
  _keys: {
    'key-1': new crypto.PrivateKey(os.readFile('sample-key-rsa.pem')),
    'key-2': new crypto.PrivateKey(os.readFile('sample-key-ecdsa.pem')),
  },

  _services: {
    'service-2': {
      keys: { 'key-1': true, 'key-2': true },
    },
  },
})

.import({
  __turnDown: 'proxy',
  __serviceID: 'proxy',
})

.pipeline('verify')
  .replaceMessage(
    msg => (
      ((
        service,
        header,
        jwt,
        kid,
        key,
      ) => (
        service = _services[__serviceID],
        service?.keys ? (
          header = msg.head.headers.authorization || '',
          header.startsWith('Bearer ') && (header = header.substring(7)),
          jwt = new crypto.JWT(header),
          jwt.isValid ? (
            kid = jwt.header?.kid,
            key = _keys[kid],
            key ? (
              service.keys[kid] ? (
                jwt.verify(key) ? (
                  msg
                ) : (
                  __turnDown = true,
                  new Message({ status: 401 }, 'Invalid signature')
                )
              ) : (
                __turnDown = true,
                new Message({ status: 403 }, 'Access denied')
              )
            ) : (
              __turnDown = true,
              new Message({ status: 401 }, 'Invalid key')
            )
          ) : (
            __turnDown = true,
            new Message({ status: 401 }, 'Invalid token')
          )
        ) : msg
      ))()
    )
  )
```

为了模块拆分，引入了 `use` 过滤器，和函数`import`、`export`。

这里需要说明的是：在使用多模块时，`import` 和 `export` 不一定是必需的，假如没有数据共享是不需要的。

## 过滤器与函数

### use 过滤器

`use` 过滤器会将事件发送到目标 pipeline 执行，与 `link` 功能类似。区别是 `use` 跨模块，并且不支持条件判断。

目标 pipeline 执行结束，`use` 过滤器会将执行结果发送给当前 pipeline 的下一个过滤器。

因此 `jwt` pipeline 的内容可以非常简洁，核心的处理逻辑都迁移到了 `jwt.js` 的 `verify` pipeline 中。

```js
.pipeline('jwt')
  .use('jwt.js', 'verify')
```

### export 函数

原来的逻辑拆分为 `proxy.js` 和 `jwt.js` 两个模块，但是两者之前是有数据共享的：

- jwt 的认证结果，`proxy.js` 模块会用来做路由执行的条件判断
- 路由决策出的服务名，`jwt.js` 会用来判断服务是否需要鉴权，并获取需要鉴权服务的秘钥。

```js
.export('proxy', {
  __turnDown: false,
  __serviceID: '',
})
```

这两个变量被从全局变量的 scope 中移出，通过 `export` 暴露使得变量成为模块间可见，同时需要指定变量所在的模块 `proxy`。

注：这里变量名前缀从原来的一个下划线变成两个下划线（从模块私有变成模块外可见）。不是强制，但是建议以这种方式命名。

### import 函数

同理，在 `jwt.js` 模块中引入变量，也需要指定定义变量的模块名。

```js
.import({
  __turnDown: 'proxy',
  __serviceID: 'proxy',
})
```

## 思考

这样便解决了模块化以及数据共享的问题，功能上做了解耦。

那么逻辑和配置呢，现在仍然是耦合在一起的。