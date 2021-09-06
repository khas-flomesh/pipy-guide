# 0x01-基本输出

[`01-hello/hello.js`](https://github.com/flomesh-io/pipy/blob/main/tutorial/01-hello/hello.js)

```javascript
pipy()

.listen(8080) // pipeline0
  .serveHTTP(
    msg => new Message(msg.body)
  )

.listen(8081) // pipeline1
  .demuxHTTP('tell-my-port')

.listen(8082) // pipeline2
  .demuxHTTP('tell-my-port')

.pipeline('tell-my-port') // pipeline3
  .replaceMessage(
    () => (
      new Message(`Hi, I'm on port ${__inbound.localPort}!\n`)
    )
  )
```

这个脚本中使用了 `serveHTTP`、`demuxHTTP`、`replaceMessage` 三个过滤器，定义了 4 个 pipeline 监听了 3 个端口。

## 全局变量

Pipy 中可以使用全局变量，这些变量在 pipeline 的上下文中可以直接访问。比如上面脚本中用到的 `__inbound` 就是内置的全局变量，此外还有 `__filename`。除了内置的全局变量，可以通过如下的方式在脚本中添加自定义的全局变量：

```js
pipy({
    _strVal: "",
    _objVal: {
        _id: 0
    }
})
```

Pipy 的某些过滤器会在“独立”的上下文中执行（后面的教程会逐一讲解），这些独立的上下文是从原上下文复制而来的。根据变量的不同类型，在复制时会有类传值和传指针的概念。

还是上面这个例子，复制得到上下文中 `_strVal` 变量值为复制发生时的当前值，并且复制后两个 pipeline 都有各自的 `_strVal` 变量，对该变量的修改互不影响。

而对 `_objVal` 这种对象类型来说，原上下文和复制得到的上下文，访问的都是统一个对象，对其所做的修改对两个 pipeline 中都是可见的。

## 过滤器

### serveHTTP 过滤器

`serveHTTP` 过滤器提供了一个 HTTP 请求的处理函数。实际上 `serveHTTP` 可以接收两种参数：一种是如介绍 pipy 控制台时脚本中用的 `new Message('Hello!\n')`，直接提供一个 `Message` 对象；另外一种就是上面脚本中的函数。区别是前者属于静态，后者属于动态的。

该函数接受 `Message` 类型的参数，返回的结果也应该是 `Message` 类型的对象，或者 `null`。

整个处理流程是先对请求负载进行解码，再调用处理函数，最后将函数处理结果编码。

```text
   ┌───────────────┐
   │               │
   │   Decoding    │
   │               │
   └───────┬───────┘
           │
           │
   ┌───────▼───────┐
   │               │
   │    Handler    │
   │               │
   └───────┬───────┘
           │
           │
   ┌───────▼───────┐
   │               │
   │   Encoding    │
   │               │
   └───────────────┘
```

### demuxHTTP 过滤器

对请求进行解码，并发送到独立 session 的 pipeline 中，最终将 pipeline 返回的内容编码。

`Message` 被发送到目标 pipeline 的 sesison 上进行处理，在创建新的 session 时会将当前上下文“复制”到新的 session 中。每个 `Message` 在目标 pipeline 上有自己独立的一套全局变量，以及每个 filter 的独立状态。

```text
  ┌──────────────┐
  │              │
  │   Decoding   │
  │              │
  └──────┬───────┘
         │
         │
         └────────────────────────────┐
                                      │
                                      │
                               ┌──────▼──────┐
                               │             │
                               │   Handler   │
                               │             │
                               └──────┬──────┘
                                      │
                                      │
         ┌────────────────────────────┘
         │
         │
  ┌──────▼───────┐
  │              │
  │   Encoding   │
  │              │
  └──────────────┘
```

这里的“复制”，与传值和传指针一样的行为。比如下面的配置，在不论在当前 pipeline 中对全局变量做修改，`demuxHTTP` 的目标 pipeline 获取到的 `_strVal` 复制发生时的当前值；而 `_objVal` 则会访问到同一个对象。

对与字符串这类的值传递，复制之后两个 pipeline 都有各自的字符串，对该变量的修改互不影响。

```javascript
pipy({
    _strVal: "",
    _objVal: {}
})
```

### replaceMessage 过滤器

这个过滤器会替换完整的信息，包括 head 和 body。接受一个 `Message` 对象，或者一个返回 `Message` 对象的函数，这个函数的输入是原始的 `Message` 对象。

## Pipeline

pipeline3 是 pipeline1 和 pipeline2 的目标 pipeline，即 pipeline1 和 pipeline2 收到请求会将请求发送到 pipeline3 上，同时创建新的 session。

尝试请求一下，发现返回值并不同。在 pipeline 3 中使用了内置的全局变量 `__inbound`，`localPort` 是监听端口。

可以理解成 `demuxHTTP` 将当前的上下文复制到了 pipeline3 中。

```text
$ curl localhost:8081
Hi, I'm on port 8081!

$ curl localhost:8082
Hi, I'm on port 8082!
```

