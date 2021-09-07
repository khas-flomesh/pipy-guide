# 0x04 负载均衡

在上一篇，我们在路由请求到上游服务的时候使用了连接池。实际情况下，上游服务会存在不只一个实例，这篇就来讲下如何将请求路由到多个实例，达到负载均衡。

这里还是将[上一篇](01-hello_zh.md)中的脚本作为上游服务。

[04-load-balancing/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/04-load-balancing/proxy.js)

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
```

## 带负载均衡的路由规则

修改原来的路由表使其支持负载均衡，Pipy 内置了负载均衡器的实现：`RoundRobinLoadBalancer`、`LeastWorkLoadBalancer`、`HashingLoadBalancer`。

三种类型的负载均衡器都提供了如下的方法：
- `select`：选择一个实例
- `deselect`：取消选择。只有 `LeastWorkLoadBalancer` 实现了这个方法。

这里我们使用 `RoundRobinLoadBalancer`。

`RoundRobinLoadBalancer` 支持两种方式的初始化：带权重和不带权重的。

```js
//non weighted
new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082'])
//weighted
new algo.RoundRobinLoadBalancer({
    '127.0.0.1:8081': 50, 
    '127.0.0.1:8082': 50
})
```

如此，我们将原理的路由规则做下调整：

```js
{
  _router: new algo.URLRouter({
    '/*'   : new algo.RoundRobinLoadBalancer(['127.0.0.1:8080']),
    '/hi/*': new algo.RoundRobinLoadBalancer(['127.0.0.1:8081', '127.0.0.1:8082']),
  })
}
```

修改之后，原来的 `algo.URLRouter.find()` 之后的结果从实例地址，变成了负载均衡器 `_balancer`，转发前需要获得实例的地址。

## 负载均衡

既然拿到了负载均衡器，只要调用 `select` 方法才能拿到实例地址。这里没有直接用 `_balancer.select()`，而是又一次使用了缓存，这个缓存的生命周期与来自下游的连接一致。这样做可以保证来自同一个下游连接的多个请求都被路由同一个上游实例。

```js
.pipeline('routing')
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

.pipeline('load-balance')
  .handleMessageStart(
    () => _target = _balancerCache.get(_balancer)
  )
  .link(
    'forward', () => Boolean(_target),
    'no-target'
  )
```

## 思考

引入了 Pipy 之后，为系统增加了路由的功能，但也在请求链路上增加了一个跨度，对跟踪监控和问题排查增加了难度。