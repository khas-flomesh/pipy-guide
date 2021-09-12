# 0x12 协议转换

上一篇的结尾我们提出这样一种场景：

> 有两个异构的系统，分别是提供 SOAP 和 JSON 的网络服务。需要根据不同的路由，讲统一客户端的请求路由的不同的系统。也就是新旧系统共存，旧系统还无法完全下线；或者新旧两个系统的交互。

代码在 [12-body-transform](https://github.com/flomesh-io/pipy/tree/main/tutorial/12-body-transform)，在 [11-throttle](https://github.com/flomesh-io/pipy/tree/main/tutorial/11-throttle) 基础上更新。

还是 `service-1` 和 `service-2` 两个服务，二者分别接受 `XML` 和 `JSON` 格式的请求。

## 配置

`transform.json`

```json
{
  "services": {
    "service-1": {
      "jsonToXml": true
    },
    "service-2": {
      "xmlToJson": true
    }
  }
}
```

引用格式抓换的配置，这里也提供 Javascript 对象与 XML 格式互转的方法

```js
(config =>

pipy({
  _services: config.services,
  _service: null,

  _obj2xml: node => (
    typeof node === 'object' ? (
      Object.entries(node).flatMap(
        ([k, v]) => (
          v instanceof Array && (
            v.map(e => new XML.Node(k, null, _obj2xml(e)))
          ) ||
          v instanceof Object && (
            new XML.Node(k, null, _obj2xml(v))
          ) || (
            new XML.Node(k, null, [v])
          )
        )
      )
    ) : [
      new XML.Node('body', null, [''])
    ]
  ),

  _xml2obj: node => (
    node ? (
      ((
        children,
        first,
        previous,
        obj, k, v,
      ) => (
        obj = {},
        node.children.forEach(node => (
          children = node.children,
          first = children[0],
          k = node.name,
          v = children.length != 1 || first instanceof XML.Node ? _xml2obj(node) : (first || ''),
          previous = obj[k],
          previous instanceof Array ? (
            previous.push(v)
          ) : (
            obj[k] = previous ? [previous, v] : v
          )
        )),
        obj
      ))()
    ) : {}
  ),
})

)(JSON.decode(pipy.load('transform.json')))
```

## 核心逻辑 

`transform.js`

```js
.import({
  __serviceID: 'proxy',
})

.pipeline('input')
  .handleSessionStart(
    () => _service = _services[__serviceID]
  )
  .link(
    'json2xml', () => Boolean(_service?.jsonToXml),
    'xml2json', () => Boolean(_service?.xmlToJson),
    'bypass'
  )

.pipeline('json2xml')
  .replaceMessageBody(
    data => (
      XML.encode(
        new XML.Node(
          '', null, [
            new XML.Node(
              'root', null,
              _obj2xml(JSON.decode(data))
            )
          ]
        ),
        2,
      )
    )
  )

.pipeline('xml2json')
  .replaceMessageBody(
    data => (
      JSON.encode(
        _xml2obj(XML.decode(data))
      )
    )
  )

.pipeline('bypass')
```

这里用到了 `link` 过滤器针对不同服务所支持的格式将数据发送到不同的 pipeline `json2xml` 和 `xml2json` 处理。

这里使用了 Pipy 的内置变量 `JSON` 和 `XML`，均提供了如下方法：

- encode：对象转换为字符串
- decode：字符串抓换为对象
- stringify：同 encode
- parse：同 decode

`XML.Node` 用于创建一个 XML 节点对象，构造参数分别是 `名字`、`属性`、`子节点`。

## 引用模块

```js
//...
.link('bypass', () => __turnDown, 'transform')
//...
.pipeline('transform')
  .use('transform.js', 'input')
//...
```

## 思考

得益于 Pipy 强大的扩展性，PipyJS 还可以解决更多场景问题，将最佳实践抽象成模块。