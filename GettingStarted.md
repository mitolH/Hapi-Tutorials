# <font id="started">获得 Hapi</font>

## 安装 Hapi

这份教程兼容`hapi v11.x.x`版本

通过下边的方法，创建一个`myproject`的文件夹：

- 运行: `npm init` 会获得提示，这个操作会为你生成一个`package.json`文件。
- 运行: `npm install -- save hapi` 这一步会安装hapi，并且保存到package.json作为项目的依赖。

就这样！现在你获得使用hapi创建一个Server服务需要的所有东西。


## 创建一个 server

最基础的server就像下边这样：

```
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.start((err) => {

    if (err) {
        throw err;
    }
    console.log(`Server running at: ${server.info.uri}`);
});
```

首先，我们需要引入hapi。接着创建一个新的hapi server对象。接着我们添加一个关联到server，传递入一个端口作为监听。最后，开启server并输出日志标志server正在运行。

当添加server的关联时，我们通常可以提供一个主机名，IP地址，或者甚至是一个Unix的套接字文件，Windows的命名管道绑定服务器。更多详细信息，查看 [API reference](https://hapijs.com/api/#serverconnectionoptions) 。

## 添加路由

现在我们已经有了一个服务器，我们将要添加一个或者两个路由来确保做一些实际的工作。让我们来看看怎么做：

```
'use strict';

const Hapi = require('hapi');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply('Hello, world!');
    }
});

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply('Hello, ' + encodeURIComponent(request.params.name) + '!');
    }
});

server.start((err) => {

    if (err) {
        throw err;
    }
    console.log(`Server running at: ${server.info.uri}`);
});
```

保存以上为 `server.js` 文件，接着用命令 `node server.js` 来启动服务器。现在，你使用浏览器访问 [http://localhost:3000](http://localhost:3000)，将会看到 `Hello, Words!`，访问 [http://localhost:3000/stimpy](http://localhost:3000/stimpy)将会看到 `Hello, stimpy!`。

注意，我们使用 URI 编码参数防止内容注入的攻击。记住，不经过编码，直接呈现用户提供的数据永远不是一种好的方法。

`method` 可以是每个有效的 HTTP 方法，HTTP 方法数组，或者*来允许所有方法。`path`参数定义的路径包括URI参数序列，可以可以是任意的字符，数字，甚至是任意匹配符。更多信息，查看 [`routing 教程`](./Routing.md)

## 创建静态页面和内容

我们已经证明可以运行一个简单 Hapi 应用的 Hello World应用程序。接下来，我们将使用一个插件 `inert` 来提供静态页面。在你开始之前，使用 **`CTRL + C`** 停止服务器。

在命令行中使用命令：`npm install --save inert` 来安装 **inert** 。这将会下载 inert 并添加到 package.json，文件记录了已经安装的包。

添加一下内容到 `server.js` 文件：

```
server.register(require('inert'), (err) => {

    if (err) {
        throw err;
    }

    server.route({
        method: 'GET',
        path: '/hello',
        handler: function (request, reply) {
            reply.file('./public/hello.html');
        }
    });
});
```

上方的 `server.register()` 命令添加了 `inert` 插件到 `Hapi` 应用。我们希望知道如果有什么错误发生，因此我们传递了一个匿名函数，如果在有错误的时候将会在 `err` 中获得，并抛出错误。这个回调函数在注册插件时候是必须的。

`server.route()` 命令注册了`/hello` 路由，通知服务器 `/hello` 接收 GET 请求并回复 `hello.html` 文件的内容。我们在注册 `inert` 时已经写入了路由的回调函数，因为我们需要确保 `inert` 在渲染静态页面之前已经注册成功。通常运行这样的代码是明智的，依赖插件的时候使用 register 的回调函数，这样你就能绝对的确定代码运行时这个插件已经存在。

`npm start` 运行服务器后在浏览器访问 [http://localhost:3000/hello](http://localhost:3000/hello)。Oh no! 我们看到了404，因为我们从未创建hello.html页面。你需要创建这个缺少的文件来解决这个错误。

在根目录创建一个叫 `public` 文件夹，并在文件夹中创建一个 `hello.html` 文件。在 `hello.html` 文件中放入以下 `HTML` 内容：```<h2>Hello World.</h2>```。在刷新浏览器的页面。应当会看到一个标题：```'Hello world.'```。

`Inert` 将会在请求的时候返回任意保存在硬盘的内容，这就导致了热加载特性。自定义 `/hello` 页面成你喜欢的样子。

更多关于静态内容如何被服务的详细信息在[静态内容服务](https://hapijs.com/tutorials/serving-files)。这个方法通常是为Web程序提供服务图像、样式表和静态页面服务。


## 使用插件

创建任何Web应用程序都需要的需求就是访问日志。让我们来使用 `good` 插件的 `good-console` 播报者来为我们的程序添加一些基础的日志信息。通常我们也需要一个基础的过滤机制。让我们来使用 `good-squeeze`，因为它已经具备我们开始所需要基础的时间类型和标记筛选。

让我们从 npm 安装这些模块开始：


```
npm install --save good
npm install --save good-console
npm install --save good-squeeze
```

接着更新 `server.js`：

```
'use strict';

const Hapi = require('hapi');
const Good = require('good');

const server = new Hapi.Server();
server.connection({ port: 3000, host: 'localhost' });

server.route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply('Hello, world!');
    }
});

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function (request, reply) {
        reply('Hello, ' + encodeURIComponent(request.params.name) + '!');
    }
});

server.register({
    register: Good,
    options: {
        reporters: {
            console: [{
                module: 'good-squeeze',
                name: 'Squeeze',
                args: [{
                    response: '*',
                    log: '*'
                }]
            }, {
                module: 'good-console'
            }, 'stdout']
        }
    }
}, (err) => {

    if (err) {
        throw err; // something bad happened loading the plugin
    }

    server.start((err) => {

        if (err) {
            throw err;
        }
        server.log('info', 'Server running at: ' + server.info.uri);
    });
});
```

现在当服务器运行的时候你会看到：
```
140625/143008.751, [log,info], data: Server running at: http://localhost:3000
```
并且如果我们在浏览器访问 [http://localhost:3000/](http://localhost:3000/)，将会看到：
```
140625/143205.774, [response], http://localhost:3000: get / {} 200 (10ms)
```
很好！这只是一个插件功能的简短示例，查看[插件教程]()获得更多信息。


## 其他的东西

`hapi` 还有很多其他的功能，教程文档中只选择了很少一部分来成文。请你在他们中选择出正确的组合。其他剩下的在 [API reference](https://hapijs.com/api) 中，像往常一样，想提问或者访问我们就到 freenode 的 [#hapi](http://webchat.freenode.net/?channels=hapi)