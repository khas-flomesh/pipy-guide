# 0x09 配置

做程序设计时，除了要考虑功能的解耦做到模块化，也需要考虑配置与逻辑的解耦。避免在逻辑代码中嵌入配置信息，这样不仅方便代码维护，也照顾到了程序的运维。

配置与逻辑分离后，才有可能进一步实现代码的复用，甚至配置的实时更新。

由于代码量的增加会占据大量篇幅，从这篇开始便不再贴出完整的代码，只列出在前一篇基础上改动的片段。完整代码通过下面连接访问。

- [09-configuration/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/09-configuration/proxy.js)
- [09-configuration/proxy.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/09-configuration/proxy.json)
- [09-configuration/jwt.js](https://github.com/flomesh-io/pipy/blob/main/tutorial/09-configuration/jwt.js)
- [09-configuration/jwt.json](https://github.com/flomesh-io/pipy/blob/main/tutorial/09-configuration/jwt.json)


## 思路

`proxy.js` 模块中的监听端口、服务鉴权及负载均衡、路由规则，还有日志数据的目标地址都可以分离成配置，保存在 `proxy.json` 中。

```
{
  "listen": 8000,
  "services": {
    "service-1": {
      "targets": [
        "127.0.0.1:8080"
      ]
    },
    "service-2": {
      "targets": [
        "127.0.0.1:8081",
        "127.0.0.1:8082"
      ]
    }
  },
  "routes": {
    "/*": {
      "service": "service-1"
    },
    "/hi/*": {
      "service": "service-2",
      "rewrite": [
        "^/hi",
        "/hello"
      ]
    }
  },
  "logURL": "http://127.0.0.1:8123/log"
}
```

`pipy` 有个 `load` 函数可以从指定文件中加载数据，返回字符串。然后通过内置变量 `JSON` 的 `decode` 函数解码成对象。

接着通过匿名函数的方式传递给原来的脚本：

```js
(config =>
    pipy({
    //...
    })
    //....
)(JSON.decode(pipy.load('proxy.json')))
```

重新调整全局变量 `_services` 和 `_router` 的初始化，Pipy 内置了 `Object` 全局变量，实现了标准库的用法。

```js
{
  _services: (
    Object.fromEntries(
      Object.entries(config.services).map(
        ([k, v]) => [
          k,
          {
            balancer: new algo.RoundRobinLoadBalancer(v.targets),
          }
        ]
      )
    )
  ),
  _router: new algo.URLRouter(
    Object.fromEntries(
      Object.entries(config.routes).map(
        ([k, v]) => [
          k,
          {
            ...v,
            rewrite: v.rewrite ? [
              new RegExp(v.rewrite[0]),
              v.rewrite[1],
            ] : undefined,
          }
        ]
      )
    )
  )
}
```

## 思考

配置与逻辑的分离，以及模块功能的支持，让脚本可以实现更多的功能。

到目前，服务治理方面还只是实现了路由及负载均衡，后面将尝试过多的功能。