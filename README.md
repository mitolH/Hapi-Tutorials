# Hapi-Tutorials
- A simple chinese hapijs tutorials
- 一个简易的中文 hapijs 教程
---
## 身份认证

这份教程兼容`hapi v11.x.x`版本

hapi的身份认证是基于`计划组`和`策略组`。

想到一种简单的身份认证方案，类似于`basic(基本数据)` 或者说 `digest(摘要)`。另外一种方案是有预先方案实例的配置。

首先，让我们来看看一个示例，怎么去使用 `hapi-auth-basic`

```
'use strict';
const Bcrypt = require('bcrypt');
const Hapi = require('hapi');
const Basic = require('hapi-auth-basic');
const server = new Hapi.Server();
server.connection({ port: 3000 });
const users = {
  john: {
        username: 'john',
        password: '$2a$10$iqJSHD.BGr0E2IxQwYgJmeP3NvhPrXAeLSaGCj6IR/XU5QtjVu5Tm',   // 'secret'
        name: 'John Doe',
        id: '2133d32a'
  }
};

const validate = function (request, username, password, callback) {
    const user = users[username];
    if (!user) {
        return callback(null, false);
    }

    Bcrypt.compare(password, user.password, (err, isValid) =   {
        callback(err, isValid, { id: user.id, name: user.name });
  });
};

server.register(Basic, (err) => {

  if (err) {
      throw err;
  }

  server.auth.strategy('simple', 'basic', { validateFunc: validate });
  server.route({
      method: 'GET',
      path: '/',
      config: {
          auth: 'simple',
          handler: function (request, reply) {
              reply('hello, ' + request.auth.credentials.name);
          }
      }
  });

  server.start((err) => {

      if (err) {
          throw err;
      }

      console.log('server running at: ' + server.info.uri);
  });
});
```
第一，我们申明了一个名为`users`的数据库，当然在本例中仅使用一个简单的对象代替。接着声明了一个验证函数，作为`hapi-auth-basic`的特性，这个函数让我们去验证用户是否拥有身份权限。


接下来，我们创建了一个计划，注册为`basic`插件，通过插件`server.auth.scheme()`注册成功。

当注册被插件之后，我们使用`server.auth.strategy()`创建了一个名为`simple`的策略指向我们的`basic`计划。同时我们传递了一个配置对象给计划，并且允许我们配置它的行为。

最后一做的一件事是告诉一条路由使用这个`simple`策略来作为身份验证。

## 计划组（Schemes）

每个计划都是一个签名为`(server, options)`的方法。`server`参数是引用的计划所加到的Server对象，`options`参数是一个配置对象，在注册某策略时使用该计划提供服务。

该方法必须返回一个至少有`authenticate`键值的对象，其他可选字段还有`payload`和`payload`。

## authenticate

`authenticate`方法的签名格式为`function (request, reply)`，并且是计划的唯一必要方法。

在签名中，`request`是`server`对象创建的`request`对象，和route的handler中可用对象是相同的对象，文档在 [API reference](http://hapijs.com/api#request-object)。

`reply`是标准的hapi的`reply`接口，它可以接收序列为`err`，`result`的参数。

如果`err`是一个非空的值，这表明在权限验证的时候发现了错误，并且这个错误将会被作为一个`reply`来返回给终端用户。建议使用`boom`来创建错误来确保可以很轻松的提供一个状态代码和信息。

`result`参数应当是一个对象，它的所有键值都是可以设置的，除非`err`参数被提供。。

如果你想在失败的时候提供更多的详情，`result`对象必须要有一个`credentials`对象属性，表示用户已经验证（或者用户试图验证的凭证），并且应当像这样调用`reply(error, null, result);`

当验证成功，应该调用`reply.continue(result)`，并且`result`是一个有`credentials`属性的对象。

此外，应当有一个`artifacts`键，其中包含了不属于用户验证信息的所有验证相关数据；

`credentials`和`artifacts`属性可以作为`request.auth`对象的一部分访问（例如在route 的handler）

## payload

`payload`方法的方法签名为`function (request, reply)`。

再一次，标准的hapi`reply`接口在这儿依然可用。通过调用`reply(error, result)`或者简易的`reply(error)`（同样也推荐使用`boom`）来传达错误信号。

通过调用`reply.continue()`不携带参数来传递成功的信号。


## response

`response`方法常规的签名为`function (request, reply)`，并且也适用标准的`reply`接口。

该方法在传递给用户之前，添加了额外的header来修饰response对象（`request.response`）。


当所有修饰完成之后，应当调用`reply.continue()`，这样response就会被传递。

如果有错误发生，不应该调用`reply(error)`，我们推荐使用`boom`来处理错误。



## Registration

相同的也使用`server.auth.scheme(name, scheme)`，注册一个schema。`name`参数是一个字符串用来识别确定的schema，这个`schema`参数是一个方法正如上文描述一样。


## Strategies

当你注册schema之后，就一种方法来使用它。这个就是策略组。

正如上边提及的，一个策略本质上就是`schema`的一个预配置的复制。

要注册一个策略，我们应当至少有一个`schema`已经注册。当`schema`注册成功之后，使用`server.auth.strategy(name, schema, [mode, options])`来注册自己的策略。

`name`参数必须是一个字符串，稍后用来识别确定的策略。`schema`也是一个字符串，是这个schema的名称，同时也是该策略的实例。


## Mode

`mode`是第一个可选的参数，可以是 `true`,`false`,`required`,`optional`或者`try`。

`mode`的默认值是 `false`，表示策略将会被注册但是不会被使用，除非你人为的调用。

如果设置`true`或者`required`，都将会自动的指定到所有不包含`auth`配置的路由中。这样的设置意味着，用户访问某个路由需要身份验证，并且是有效的验证信息，否则会获得一个错误。

如果`mode`设置为`'optional'`，这个策略也会被应用到缺少`auth`的路由中，但在这种情况下的路由不需要身份验证。验证数据是可选的，但是如果提供的话必须是有效的。

`mode`的最后一个值时`try`，同样也是自动使用到缺少`auth`配置的路由中。和`optional`的不同之处在于，如果用户提供的验证信息无效的时候，仍然可以到达路由的handler。


## Options

最后一个参数是`options`，这个参数将会直接传递到声明的schema


## 设置一个默认的策略（Setting a default strategy）

正如前边所提到的，`mode`参数能够和`server.auth.strategy()`使用来设置一个默认的策略。通常也可以使用`server.auth.default()`来明显的设置一个默认策略。


这个函数接收一个字符串（默认策略的名称）作为参数，或者说一个对象和路由handler中的[auth options](#route-configuration)一样的格式。

需要注意的是：任何在`server.auth.default()`之前被调用的路由都不会被默认的策略影响。因此你想要保证所有的路由都有默认的策略启用，你都必须在你所有的路由前调用`server.auth.default()`或者在注册策略的时候设置默认的模式。

## <font id="route-configuration">路由配置（Route configuration）</font>

验证通常可以使用`config.auth`参数配置在路由中。如果设置为`false`的话，该路由中的验证将不可用。

通常设置为使用的策略名称字符串，或者一个对象包括`mode`,`strategies`和`payload`参数。

`mode`参数可以设置为`'required'`, `'optional'`, 或者 `'try'`，他们的作用和注册策略时是一样的。


当你指定一个策略的时候，应当设置`strategy`属性，并且应当时一个策略名称的字符串。当指定多个策略的时候，这个属性名称应该为`strategies`，并且应当是一个所有策略名称字符串组成的数组。策略组将会被顺序执行，除非直到其中一个成功或者全部都失败。

最后，`payload`参数可以设置为 `false` 表明，这个装载（该怎么翻译啊-_-#？payload）不能被验证，`'require'`或者 `true`表示必须被验证，`'optional'`表示如果客户端有装载验证信息，则验证必须校验。

`payload`参数只能被用到一个策略中，支持`payload`方法的schema中。