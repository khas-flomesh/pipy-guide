# 0x05 日志

Pipy 的代理路由在原来的服务链路上增加了一次网络调用，让分布式监控进一步复杂。日志是最常用的监控手段之一，这次就来介绍下日志记录的脚本实现。

[05-logging/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/05-logging/proxy.js)

```js
pipy({
  _router: new algo.URLRouter({
    '/*'   : new algo.RoundRobinLoadBalancer(['127.0.0.1:8080']),
    '/hi/*': new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082']),
  }),

  _g: {
    connectionID: 0,
  },

  _connectionPool: new algo.ResourcePool(
    () => ++_g.connectionID
  ),

  _balancer: null,
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
      _balancer = _router.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .link(
    'load-balance', () => Boolean(_balancer),
    '404'
  )
  .fork('log-response')

.pipeline('load-balance')
  .handleMessageStart(
    () => _target = _balancerCache.get(_balancer)
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

[05-logging/mock.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/05-logging/mock.js)

```js
pipy()

.listen(8123)
  .serveHTTP(
    msg => console.log(msg.body.toString())
  )
  .dummy()
```

对请求的日志记录都关注哪些点？请求和响应的基础信息（如 host、path、header、method 之类的请求/响应头）、请求影响的时间、耗时、地址、端口等。其中耗时是通过收到响应的时间与请求发出的时间计算所得，这就意味着需要使用“值传递类型”的环境变量，同时记录的操作在独立的 session 中完成。

其次日志的记录不能影响路由这个主流程，需要在旁支 pipeline 中完成。Pipy 内置提供的 `fork` 过滤器恰好可以满足这一需求。


## 过滤器

### fork

`fork` 过滤器将消息会发送输入的拷贝到另一个 pipeline，并在在独立的 session 执行。同时将输入**原封不动**的输出，即 `fork` 不会对当前上下文以及消息做任何的修改；同时 `fork` 会继续使用原来的上下文，因此<u>对全局变量的修改会体现在原来的 session 中</u>（`routing` pipeline 的 session）。

修改 `routing` pipeline，在路由操作的前后使用 `fork` 将请求和响应的拷贝发送到 `log-request` 和 `log-response` 两个旁支 pipeline 中执行：
- `log-request`：记录请求到全局变量中
- `log-response`：记录响应的相关信息，结合全局变量中的请求信息，组装日志消息并发送到 `log-send` pipeline。这里使用了 `merge` 过滤器，见下文。

```js
.pipeline('routing')
  .fork('log-request')
  .handleMessageStart(
    msg => (
      _balancer = _router.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .link(
    'load-balance', () => Boolean(_balancer),
    '404'
  )
  .fork('log-response')
  
.pipeline('log-request')
  .handleMessageStart(
    () => _requestTime = Date.now()
  )
  .decompressHTTP()
  .handleMessage(
    '256k',
    msg => _request = msg
  )
```

### merge

`log-response` 到 `log-send` 使用了 `merge` 进行连接。`merge` 与 `fork` 同样不会对输入进行任何修改，同时会共用原来（`log-response`）的 session。

```js
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
```

### pack

`pack` 过滤器用来完成类似批处理的操作：将多个消息压缩成一个。

使用格式：

```
.pack(int, {
    timeout: int, // seconds, default 0
    vacancy: int // 1 or 0, default 0.5
}
```

比如这里我们将 1000 个消息压缩成 1 个、超时时间设置为 5s，意思是 1000 和 5s 任何一个条件先达到就会将压缩好的消息发出。

这里并没有设置 `vacancy`，意思是当单个消息不足 4kb 的整数倍时是否保留冗余空间。设置为 `1` 不会压缩。

```js
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

`pack` 过滤器输出的是 `Data` 类型的数据，与请求/响应的消息体一致。发送到上游的时所需的消息还缺少消息头，因此使用了 `replaceMessageStart` 写入一个 `MessageStart` 对象携带消息头。

## 测试

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
    "path": "/hi",
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
  "reqTime": 1631152062810,
  "resTime": 1631152062810,
  "endTime": 1631152062810,
  "remoteAddr": "127.0.0.1",
  "remotePort": 51574,
  "localAddr": "127.0.0.1",
  "localPort": 8000
}
```

## 思考

这次的脚本使用了多个新的过滤器，这些过滤器跨越了不同的 pipeline，对 session 和上下文有了不同的操作。

