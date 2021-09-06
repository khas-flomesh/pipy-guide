# 0x03-连接池

在上一篇 [路由](02-routing_zh.md) 中我们通过了 `muxHTTP` 完成向上游的请求转发。假如上游只有一个实例，下游的多个请求都在同一个连接上完成。但是 HTTP 1 的多路复用下响应也是按照请求的顺序返回给下游，这就意味着假如其中某个请求的延迟比较大，会影响到后面的请求。

此时我们需要考虑使用连接池，来增加到上游的连接数。

[`03-connection-pool/proxy.js`](https://github.com/flomesh-io/pipy/blob/main/tutorial/03-connection-pool/proxy.js)

```js
pipy({
  _router: new algo.URLRouter({
    '/*'   : '127.0.0.1:8080',
    '/hi/*': '127.0.0.1:8081',
  }),

  _g: {
    connectionID: 0,
  },

  _connectionPool: new algo.ResourcePool(
    () => ++_g.connectionID
  ),

  _target: '',
  _targetCache: null,
})

.listen(8000)
  .handleSessionStart(
    () => (
      _targetCache = new algo.Cache(
        // k is a target, v is a connection ID
        (k  ) => _connectionPool.allocate(k),
        (k,v) => _connectionPool.free(v),
      )
    )
  )
  .handleSessionEnd(
    () => (
      _targetCache.clear()
    )
  )
  .demuxHTTP('routing')

.pipeline('routing')
  .handleMessageStart(
    msg => (
      _target = _router.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .link(
    'forward', () => Boolean(_target),
    '404'
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
```

在上一篇脚本的基础上增加了几个全局变量：
- _g：一个对象，包含了一个整形的 `connectionID`。由于是对象，<u>`_g` 是 pipy scope 的</u>。
- _connectionPool：`algo.ResourcePool` 资源池类型的对象。<u>这个变量同样是 pipy scope 的</u>。
- _targetCache：默认值是 null。在 `handleSessionStart` 中初始化为 `algo.Cache` 缓存类型，<u>这个变量是 session scope 的，可以理解为 connection scope，生命周期与 connection 相同。</u>

## 资源池 `algo.ResourcePool`

内置的 `algo.ResourcePool` 类型，接受一个函数作为初始化构造参数，这个函数就是一个分配器。

方法：
- `allocate` 方法的参数为资源池的 name，返回资源。假如没有可用资源，则会调用分配器分配一个，并放入池中。
- `free` 方法用来释放资源。

## 缓存 `algo.Cache` 

内置的 `algo.Cache` 类型，接受 2 个参数作为初始化构造参数：分配函数、回收函数：
- 分配函数：缓存无法命中时调用，用来获取新的资源。
- 回收函数：元素删除或者缓存销毁时调用，释放资源。

方法：
- `get`：从缓存中获取元素
- `set`：向缓存中增加元素
- `remove`：删除元素
- `clear`：清空缓存

### 缓存的初始化和清理

```js
.listen(8000)
  .handleSessionStart(
    () => (
      _targetCache = new algo.Cache(
        // k is a target, v is a connection ID
        (k  ) => _connectionPool.allocate(k),
        (k,v) => _connectionPool.free(v),
      )
    )
  )
  .handleSessionEnd(
    () => (
      _targetCache.clear()
    )
  )
```
### 缓存元素的获取

```js
.pipeline('forward')
  .muxHTTP(
    'connection',
    () => _targetCache.get(_target)
  )
```

## 思考

下游使用<u>同一个连接</u>通过 pipy 多次访问同一个上游服务，使用的都是同一个缓存。只有在链接断开后，才会清空缓存，释放到上游的连接。

开头我们将场景设定在了上游服务只有一个实例，假如上游有多个实例，如何实现负载均衡？能否保证同一个下游连接的多次请求，能够路由到同一个实例上。