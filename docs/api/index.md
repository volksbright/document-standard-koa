# 安装

  Koa 需要 __Node.js v7.6.0__ 或更高版本，来支持 ES2015 和 async 函数。

  使用自己常用的版本管理工具，快速地安装一个可以支持的 Node.js 版本：

```bash
$ nvm install 7
$ npm i koa
$ node my-koa-app.js
```

# 应用程序(application)

Koa 应用程序是一个对象，通过把一系列中间件函数(middleware function)组合成一个处理函数，这样在接收到请求时，这些函数会以类似栈的方式去顺次执行。Koa 类似于你可能遇到过的许多其他中间件系统，例如 Ruby Rack、Connect 等，但是 Koa 做出了一个关键的设计决策，在其低级别的中间件层提供高级语法糖。这提高了互操作性、健壮性，这使得编写中间件函数更加愉快。

Koa 中包含了内容协商、缓存新鲜度、代理支持和重定向等常见任务的方法。尽管提供了相当多的有用方法，但 Koa 仍然保持较小的占用空间，这是因为 Koa 自身没有捆绑中间件函数。

这里是一个必须出现的 hello world 应用程序：

<!-- runkit:endpoint -->
```js
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

## 级联(cascade)

Koa 中间件以更加传统的方式进行级联调用，你可能已经使用过类似的工具 - 在你原先使用 Node.js 回调实现时，很难做到用户友好。然而，通过 async 函数，我们可以实现“正确的”中间件。对比 Connect 的实现，Koa 只是通过一系列函数传递控制，等待其中一个函数返回，Koa 调用“下游”，然后控制流回到“上游”。

下面的示例响应 "Hello World"，但是首先请求流经 `x-response-time` 和 `logging` 中间件，以标记请求何时开始，然后通过 `response` 中间件产生控制。当中间件调用 next() 时，该函数会挂起并将控制权，传递给下一个定义的中间件。在没有更多的中间件要执行下游后，堆栈将展开逐个调用，每个中间件将恢复执行其上游行为。

<!-- runkit:endpoint -->
```js
const Koa = require('koa');
const app = new Koa();

// logger

app.use(async (ctx, next) => {
  await next();
  const rt = ctx.response.get('X-Response-Time');
  console.log(`${ctx.method} ${ctx.url} - ${rt}`);
});

// x-response-time

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});

// response

app.use(async ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

## 设置

应用程序设置是 `app` 实例上的属性，目前支持以下内容：

  - `app.env` 默认是 __NODE_ENV__ 或 "development"
  - `app.keys` 由 `signed cookie` key 构成的数组
  - `app.proxy` 在正确的代理头字段将被信任时
  - `app.subdomainOffset` 要忽略的 `.subdomains` 偏移量，默认是 2
  - `app.proxyIpHeader` 代理 IP 头，默认是 `X-Forwarded-For`
  - `app.maxIpsCount` 从代理 IP 头读取的最大 IPs，默认是 0（表示无穷大）

你可以将设置，传递给构造函数：

  ```js
  const Koa = require('koa');
  const app = new Koa({ proxy: true });
  ```
或者动态地设置：
  ```js
  const Koa = require('koa');
  const app = new Koa();
  app.proxy = true;
  ```

## app.listen(...)

一个 Koa 应用程序不是只对应到一个 HTTP 服务器。可以将一个或多个 Koa 应用程序挂载到一起，以形成在单个 HTTP server 时运行多个应用程序。

创建并返回一个 HTTP 服务器，将给定的参数传递给 `Server#listen()`。这些参数记录在 [nodejs.org](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback) 上。下面是一个绑定到端口 `3000` 的 Koa 应用程序：

```js
const Koa = require('koa');
const app = new Koa();
app.listen(3000);
```

`app.listen(...)` 方法只是以下内容的语法糖：

```js
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
```

这表示你可以在 HTTP 和 HTTPS 或多个地址上启动相同的应用程序：

```js
const http = require('http');
const https = require('https');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
https.createServer(app.callback()).listen(3001);
```

## app.callback()

返回一个回调函数，用于配合 `http.createServer()` 方法处理请求。你也可以使用这个回调函数在 Connect/Express 应用程序中挂载你的 Koa 应用程序。

## app.use(function)

将给定的中间件函数添加到当前应用程序中。`app.use()` 会返回 `this`，因此这里是可以链式调用的。
```js
app.use(someMiddleware)
app.use(someOtherMiddleware)
app.listen(3000)
```
与下面相同
```js
app.use(someMiddleware)
  .use(someOtherMiddleware)
  .listen(3000)
```

查看 [中间件(middleware)](https://github.com/koajs/koa/wiki#middleware) 了解更多信息。

## app.keys=

设置 `signed cookie` key。

这些 key 会传递给 [KeyGrip](https://github.com/crypto-utils/keygrip)，然而你也可以传递自己的 `KeyGrip` 实例。例如，可以接受以下传递方式：

```js
app.keys = ['OEK5zjaAMPc3L6iK7PyUjCOziUH3rsrMKB9u8H07La1SkfwtuBoDnHaaPCkG5Brg', 'MNKeIebviQnCPo38ufHcSfw3FFv8EtnAe1xE02xkN1wkCV1B2z126U44yk2BQVK7'];
app.keys = new KeyGrip(['OEK5zjaAMPc3L6iK7PyUjCOziUH3rsrMKB9u8H07La1SkfwtuBoDnHaaPCkG5Brg', 'MNKeIebviQnCPo38ufHcSfw3FFv8EtnAe1xE02xkN1wkCV1B2z126U44yk2BQVK7'], 'sha256');
```

出于安全考虑，请确保 key 足够长且随机。

这些密钥可以轮换并在使用 `{ signed: true }` 选项签发 cookie 时使用：

```js
ctx.cookies.set('name', 'tobi', { signed: true });
```

## app.context

`app.context` 是创建 `ctx` 时的原型对象。你可以通过编辑 `app.context` 向 `ctx` 添加其他属性。这一机制对于向 `ctx` 添加属性或方法，以在整个 app 中使用很有帮助，可能会更高效（没有中间件）和更容易（更少的 `require()`），但这一机制的代价是更多地依赖 `ctx`，可以被认为是一种反模式。

例如，从 `ctx` 中引用数据库：

```js
app.context.db = db();

app.use(async ctx => {
  console.log(ctx.db);
});
```

注意：

- `ctx` 上的许多属性是使用 getter、setter 和 `Object.defineProperty()` 定义的。你只能通过在 `app.context` 上使用 `Object.defineProperty()` 来编辑这些属性（不推荐）。请查看 https://github.com/koajs/koa/issues/652。
- 当前挂载的应用程序，使用其父级的 `ctx` 和设置。因此，挂载的应用程序实际上只是一组中间件。

## 错误处理

默认情况下，除非 `app.silent` 是 `true`，否则将所有错误输出到 stderr。当 `err.status` 是 `404` 或 `err.expose` 是 `true` 时，默认错误处理程序也不会输出错误。要执行自定义错误处理逻辑，例如集中日志，可以添加一个 "error" 事件监听函数：

```js
app.on('error', err => {
  log.error('server error', err)
});
```

如果在 req/res 顺次调用中出现错误，并且_没有可能_响应客户端，这时还会传递 `Context` 实例：

```js
app.on('error', (err, ctx) => {
  log.error('server error', err, ctx)
});
```

在出现错误，并且_仍然可以_响应客户端，也就是没有数据写入 socket，Koa 将适当地响应一个 500  "Internal Server Error(服务器内部错误)"。在这两种中的任何一种情况下，出于记录日志目的，都会发出应用程序级别的 "error"。
