# 0x06 路径重写

前面我们在配置路由时候，用到了前缀匹配 `/hi/*`。实际场景中（尤其是作为网关处理南北向流量时），这个路径更多是用来标识上游，而上游服务的并不需要此路径，甚至有时需要换成其他的路径。这便是路径重写（Path Rewriting）功能需要处理的。

[06-path-rewriting/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/06-path-rewriting/proxy.js)

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
  .link(
    'load-balance', () => Boolean(_service),
    '404'
  )
  .fork('log-response')

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
```

## 配置

既然这个路径只是用来做上游路由匹配，那就要在 `_router` 上做文章了。在 [负载均衡](04-load-balancing_zh.md#带负载均衡的路由规则) 那篇时，路径匹配到的负载均衡器。既然要实现路径重写，首先我们抽象出新的 `Service` 对象，相当于负载均衡器上再封装一次，并在路由决策时修改消息头中的 `path` 为指定路径。

*当然，这里可以 `Service` 类型不是必须，这样做只是将路由和负载均衡器解耦，方便后面的扩展。*

增加一个全局变量 `_services`，并修改路由规则。

```js
{
    //...
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
    //...
}
```

既然路由规则修改了，决策的逻辑也要相应的调整，别忘记了我们要做路径重写。

## 路由决策

之前讲路由的时候我们介绍过[HTTP 消息解码时的各种事件](02-routing_zh.md#路由决策)，`handleMessageStart` 会在接收到 `MessageStart` 事件时执行，**并会将事件传递下去**。

因此我们可以对事件对象进行修改，完成路径重写。

```js
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
```

## 结果

从 [日志记录](05-logging_zh.md) 中 mock 服务的日志中，可以看到效果。请求的记录是发生在路由决策后，因此可以看到重写后的路径。

```js
{
  "req": {
    "protocol": "HTTP/1.1",
    "headers": {
      "host": "localhost:8000",
      "user-agent": "HTTPie/2.4.0",
      "accept-encoding": "gzip, deflate",
      "accept": "*/*",
      "connection": "keep-alive"
    },
    "method": "GET",
    "path": "/hello", // HERE
    "body": ""
  },
  "res": {
    "protocol": "HTTP/1.1",
    "headers": {
      "connection": "keep-alive"
    },
    "status": 200,
    "statusText": "OK",
    "body": "Hi, I'm on port 8081!\n"
  },
  "target": "127.0.0.1:8081",
  "reqTime": 1631151888233,
  "resTime": 1631151888234,
  "endTime": 1631151888234,
  "remoteAddr": "127.0.0.1",
  "remotePort": 50810,
  "localAddr": "127.0.0.1",
  "localPort": 8000
}
```