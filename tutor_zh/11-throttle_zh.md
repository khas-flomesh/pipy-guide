# 0x11 流量控制

[上一篇](10-ban_zh.md)中我们分析了如何通过模块的方式为脚本增加黑白名单，这一次是流量控制。

代码在 [11-throttle](https://github.com/flomesh-io/pipy/tree/main/tutorial/11-throttle)，与 [10-ban](https://github.com/flomesh-io/pipy/tree/main/tutorial/10-ban) 对比。

继续沿用模块添加的方式：

```js
.pipeline('routing')
  //...
  .link('bypass', () => __turnDown, 'ban')
  .link('bypass', () => __turnDown, 'throttle') //new module
  .link('bypass', () => __turnDown, 'jwt')
  //...

.pipeline('throttle')
  .use('throttle.js', 'input')
  
//...
```

## 配置

流量控制的统计维度有很多种：服务、路径、来源 ip、甚至 HTTP 请求头，以及前面的各种组合。这里就用最简单，用服务名维度来控制（下文会介绍如何控制统计的维度）。

`throttle.json` 中配置配置服务的流量限额，

```json
{
  "services": {
    "service-1": {
      "rateLimit": 1000
    }
  }
}
```

脚本中用流量控制的配置初始化全局变量 `_services`：

```js
(config =>

pipy({
  _services: config.services,
  _rateLimit: undefined,
})

//...

)(JSON.decode(pipy.load('throttle.json')))
```

## 过滤器

流控的核心逻辑：

```js
.pipeline('input')
  .handleSessionStart(
    () => _rateLimit = _services[__serviceID]?.rateLimit
  )
  .link(
    'throttle', () => Boolean(_rateLimit),
    'bypass'
  )

.pipeline('throttle')
  .tap(
    () => _rateLimit,
    () => __serviceID,
  )
```

这通过主脚本的变量 `__serviceID` 确定目标服务，从全局变量中 `__services` 获取到流控的配置，输入到 pipeline `throttle` 进行处理。

`throttle` 使用了 `tap` 过滤器，`tap` 用于消息请求数和数据的流量控制，语法如下。

```js
.tap(quota[, account])
```

- `quota`：可以是整数、字符串或者函数。如果是整数，比如 `10000`，则表示请求数限制 10000/s；如果是字符串，比如 `100k`，则表示请求数量限制 100k/s；如果是函数，<u>每隔 5s 调用一次该函数更新</u>，使用函数的结果作为统计值。
- `account` ：可以是字符串或者函数，表示流控的统计维度。如果没有指定，则会使用 `Session` 的account，即整个 Session 作为统计维度。

这里重点说一下 `account`，可能会有点复杂。在 `tap` 过滤器中有一个 `AccountManager` 用于维护多个 `Account`（参数 `account` 是函数时，会产生动态的 `Account`）。`Account` 都有名字，这个名字，就是 `account` 指定的字符串本身或者函数的计算结果。假如未指定，`AccountManager` 则会创建一个无名的 `Account`，即使 Session 的 `Account`。

一个 `Pipeline`，可以有多个 `Session`。`Session` 的数量，决定了并发的能力。

看下测试。

## 测试

我们使用 [wrk](https://github.com/wg/wrk) 来模拟请求。

### 吞吐最高为 1；连接、并发数为 1

首先，不改动代码，将 `throttle.json` 中 `rateLimit` 设置为 1。意思是 `service-1` 的吞吐最高为 1。

同样的并发和连接数，执行的时间越长，结果越精准。

符合预期。

```shell
$ wrk -t1 -c1 -d 5s http://localhost:8000
Running 5s test @ http://localhost:8000
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   699.56ms  472.81ms   1.00s    66.67%
    Req/Sec     3.17      4.54     9.00     66.67%
  6 requests in 5.05s, 372.00B read
Requests/sec:      1.19
Transfer/sec:      73.71B

$ wrk -t1 -c1 -d 30s http://localhost:8000
Running 30s test @ http://localhost:8000
  1 threads and 1 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   942.30ms  236.14ms   1.01s    93.55%
    Req/Sec     0.87      2.22     9.00     93.55%
  31 requests in 30.05s, 1.88KB read
Requests/sec:      1.03
Transfer/sec:      63.95B
```

### 吞吐最高为 1；连接，并发数为5

吞吐量为 5，符合预期

```shell
$ wrk -t5 -c5 -d 5s http://localhost:8000

$ wrk -t5 -c5 -d 3s http://localhost:8000

```

### 吞吐量为 1；连接、并发数为5；不设置 `account`

还是使用上面的配置，但是这次改动下代码。注释掉 `throttle.js` 中的 `tap` 过滤器的 `account` 参数。

```js
.pipeline('throttle')
  .tap(
    () => _rateLimit,
    //() => __serviceID,
  )
```

吞吐量最高为 5，符合预期。5 个连接，`Session` 最多为 5。每个 `Session` 的吞吐为 1。

```shell
$ wrk -t5 -c5 -d 5s http://localhost:8000

$ wrk -t5 -c5 -d 3s http://localhost:8000

```

## 思考

我们使用 `tap` 过滤器以独立模块的方式实现了流量控制。实际上的原理是，每次有请求进来 `Pipeline` 都比较记录值与 `quota`，记录值是每次请求进行 +1 的操作。

