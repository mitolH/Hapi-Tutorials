# <font id="caching">缓存</font>

## 客户端缓存

这份教程兼容`hapi v11.x.x`版本

HTTP协议提供几种不同的HTTP头来控制浏览器怎么缓存资源。通过查看[Google开发者指南](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)或者其他资料来选择合适你场景的一种。本教程仅提供预览怎么使用这些技术在hapi中。


## <font id="cache-control">缓存控制</font>

`缓存控制`头告诉浏览器和其他中间缓存资源是否可缓存以及缓存多久时间。例如，`Cache-Control:max-age=30, must-revalidate, private`表示浏览器可以缓存相关资源30秒，`private`意味着不能被中间缓存资源缓存，只能被浏览器缓存。`must-revalidate`表示有效期过期之后必须要再次从服务器请求资源。

来让我们看看在`hapi`中怎么设置这些头：

```
server.route({
    path: '/hapi/{ttl?}',
    method: 'GET',
    handler: function (request, reply) {

        const response = reply({ be: 'hapi' });
        if (request.params.ttl) {
            response.ttl(request.params.ttl);
        }
    },
    config: {
        cache: {
            expiresIn: 30 * 1000,
            privacy: 'private'
        }
    }
});
```
*实例一* 设置`Cache-Control`头

示例一已经举例说明，可以通过[response对象](http://hapijs.com/api#response-object)提供的`ttl(msec)`进行重写`expiresIn`属性。

查看[`route-options`](http://hapijs.com/api#route-options)获取更多信息关于通用缓存配置选项。


## 最后修改Last-Modified

在某些场景中服务器能提供信息，当资源被最后修改后。当使用`inert`插件来分发静态内容，在每次反馈载体中都会添加`Last-Modified`头。

当response中设置了`Last-Modified`的头，hapi会和客户端传过来的`If-Modified-Since`头进行对比，从而确定是否应当返回状态码304（远程资源未修改）。

假设`lastModified`是一个Date对象，你可以设置头通过`response`对象接口，从下边的实例二可见。
```
   reply(result).header('Last-Modified', lastModified.toUTCString());
```
*实例二* 设置 Last-Modified header

在本教程的最后一个章节有更多的使用[`Last-Modified`](#client-and-server-caching)的例子。


## ETag

ETag是`Last-Modified`的替代，服务器提供一个token（通常是资源的校验和）而不是最后修改的时间戳。浏览器会在下一次请求的时候使用这个token设置`if-None-Match`头。服务器把header的值和新的ETag校验和进行比较，如果一样的话response会返回一个304状态码。

你仅仅需要在hadler中通过`etag(tag, options)函数`设置ETag。
```
   reply(result).etag('xxxxxxxxx');
```
*实例三* 设置ETag头

查看文档了解更多[`response对象`](http://hapijs.com/api#response-object)下关于`etag`更多参数和可用配置的详细信息。


## 服务器端缓存

hapi提供强大的服务器端缓存，通过[`catbox`](https://www.github.com/hapijs/catbox)来很方便的使用缓存。本教程章节将会帮助你理解`catbox`是怎样被使用到hapi中的和怎样才能使用它。

Catbox 有两个接口：[`client`](#catbox-client)和[`policy`](#catbox-policy)。


### <font id="catbox-client">Client</font>

Client是一个低版本的接口允许你设置/获取键值对。可以使用以下可用适配器来初始化：（[`Memory`](https://github.com/hapijs/catbox-memory), [`Redis`](https://github.com/hapijs/catbox-redis), [`mongoDB`](https://github.com/hapijs/catbox-mongodb), [`Memcached`](https://github.com/hapijs/catbox-memcached), 或者 [`Riak`](https://github.com/DanielBarnes/catbox-riak)）。

Client is a low-level interface that allows you set/get key-value pairs. It is initialized with one of the available adapters: (Memory, Redis, mongoDB, Memcached, or Riak).

hapi always initialize one default client with memory adapter. Let's see how we can define more clients.

```
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server({
    cache: [
        {
            name: 'mongoCache',
            engine: require('catbox-mongodb'),
            host: '127.0.0.1',
            partition: 'cache'
        },
        {
            name: 'redisCache',
            engine: require('catbox-redis'),
            host: '127.0.0.1',
            partition: 'cache'
        }
    ]
});

server.connection({
    port: 8000
});
```
*实例四*声明catbox客服端

在实例四种，我们声明了两个catbox客户端：`mongoCache`和`redisCache`。包括hapi创建的默认内存缓存，共有三种可用的缓存客户端。在注册一个新的缓存客户端的时候可以通过删除`name`属性来修改默认的客户端。`partition`告诉适配器缓存被怎么命名（'catbox'是默认值）。当使用mongoDB的时候是数据库的名词，当使用redis的时候是使用的的键前缀。


### <font id="catbox-policy">Policy</font>

`Policy`是一个比`Client`更高版本的接口。假设我们正在处理一些比添加两个数字更加复杂的事情，并且我们希望缓存这些结果。`server.cache`创建了一个新的`policy`，稍后会被使用到路由的handler中。

```
const add = function (a, b, next) {

    return next(null, Number(a) + Number(b));
};

const sumCache = server.cache({
    cache: 'mongoCache',
    expiresIn: 20 * 1000,
    segment: 'customSegment',
    generateFunc: function (id, next) {

        add(id.a, id.b, next);
    },
    generateTimeout: 100
});

server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        const id = request.params.a + ':' + request.params.b;
        sumCache.get({ id: id, a: request.params.a, b: request.params.b }, (err, result) => {

            if (err) {
                return reply(err);
            }
            reply(result);
        });
    }
});
```
*实例五* 使用 `server.cache`


`cache`告诉hapi那个客户端将会被使用到。

`sumCache.get`函数的第一个参数是一个带有强制id键的对象，是一个唯一标识符字符串。

为了分区，`segment`允许你分割你统一客户端的缓存。如果想要缓存结果通过不同的方法，通常不希望将结果混合在一起。在`mongoDB`适配器中，`segment`表现为一个collection；在`redis`中是在`partition`选项前加一个前缀。

当`server.cache`在插件中调用时`segment`的默认值将会是`!pluginName`。当创建[`server methods`](#server-methods)的时候，`segment`的值将会是`#methodName`。如果你使用多个政策（policies）共享同一个`segment`，有一个[`shared`](http://hapijs.com/api#servercacheoptions)配置参数可以使用。


## <font id="server-methods">Server methods</font>

但它可以得到比这更好的效果！（But it can get better than that!）在95%的情景下你都将使用`server methods`来缓存，因为这样减少引用到最低。让我们使用`server methods`来编写实例五：

```
const add = function (a, b, next) {

    return next(null, Number(a) + Number(b));
};

server.method('sum', add, {
    cache: {
        cache: 'mongoCache',
        expiresIn: 30 * 1000,
        generateTimeout: 100
    }
});

server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        server.methods.sum(request.params.a, request.params.b, (err, result) => {

            if (err) {
                return reply(err);
            }
            reply(result);
        });
    }
});
```
*实例六* 通过server methods使用缓存


[`server.method()`](http://hapijs.com/api#servermethodname-method-options)自动为我们创建一个新的`policy`通过`segment: '#sum'`。通常唯一的对象id（缓存键）被自动的产生从参数中。对于更复杂的参数来说，你应当提供自己的`generateKey`函数依据参数创建唯一的id。查看[server methods教程](http://hapijs.com/tutorials/server-methods)获取更多信息。


## 接下来是什么？

- 浏览[catbox policy 选项](https://github.com/hapijs/catbox#policy)，并且额外注意`staleIn`,`staleIn`和`generateTimeout`，利用这些挖掘`catbox`缓存的所有潜在能力。

- 查看更多的server methods教程实例。


## <font id="client-and-server-caching">Client and Server caching</font>

Catbox Policy 提供了两个参数`cached`和`report`来提供一些详情当一个对象被缓存。

在这种情景下我们使用 `cached.stored` 邮戳来设置 `last-Modified` header。

```
server.route({
    path: '/add/{a}/{b}',
    method: 'GET',
    handler: function (request, reply) {

        server.methods.sum(request.params.a, request.params.b, (err, result, cached, report) => {

            if (err) {
                return reply(err);
            }
            const lastModified = cached ? new Date(cached.stored) : new Date();
            return reply(result).header('last-modified', lastModified.toUTCString());
        });
    }
});
```
*实例七*服务器和客户端缓存一起使用