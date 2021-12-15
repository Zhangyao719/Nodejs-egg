# Nodejs egg框架

## Egg与Koa的关系

Koa 是一个非常优秀的框架，然而对于企业级应用来说，它还比较基础。

Egg 选择了 Koa 作为其基础框架，在它的模型基础上，进一步对它进行了一些增强。

### 扩展

```js
// app/extend/context.js
module.exports = {
  get isIOS() {
    const iosReg = /iphone|ipad|ipod/i;
    return iosReg.test(this.get('user-agent'));
  },
};
```

在 Controller 中，我们就可以使用到刚才定义的这个便捷属性了：

```js
// app/controller/home.js
exports.handler = ctx => {
  ctx.body = ctx.isIOS
    ? 'Your operating system is iOS.'
    : 'Your operating system is not iOS.';
};
```

### 插件

 Koa 中，经常会引入许许多多的中间件来提供各种各样的功能，例如引入 [koa-bodyparser](https://github.com/koajs/bodyparser) 来解析请求 body。**而 Egg 提供了一个更加强大的插件机制，让这些独立领域的功能模块可以更加容易编写。**

一个插件可以包含

- extend：扩展基础对象的上下文，提供各种工具类、属性。
- middleware：增加一个或多个中间件，提供请求的前置、后置处理逻辑。
- config：配置各个环境下插件自身的默认配置项。

## 安装

```bash
$ npm i egg-init -g //全局安装egg
$ mkdir egg-example && cd egg-example     // 创建"egg-example"文件夹,并进入该文件夹
$ npm init egg --type=simple / --type=ts  // 简单模式/ts模式
$ npm i 
```

## 快速入门

我们推荐直接使用脚手架，只需几条简单指令，即可快速生成项目:

```shell
$ mkdir egg-example && cd egg-example
$ npm init egg --type=simple
$ npm i
```

### 编写Controller

```js
// app/controller/home.js
const Controller = require('egg').Controller;

class HomeController extends Controller {
  async index() {
    const { ctx } = this;
    ctx.body = 'hi, egg';
  }
}

module.exports = HomeController;

```

配置路由映射：

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/', controller.home.index); // 命名空间, egg约定
};
```

### 静态资源

Egg 内置了 [static](https://github.com/eggjs/egg-static) 插件，线上环境建议部署到 CDN，无需该插件。

static 插件默认映射 `/public/* -> app/public/*` 目录

此处，我们把静态资源都放到 `app/public` 目录即可：

```
app/public
├── css
│   └── news.css
└── js
    ├── lib.js
    └── news.js
```

### 模板渲染

```shell
$ npm i egg-view-nunjucks --save
```

开启插件：

```js
// config/plugin.js
exports.nunjucks = {
  enable: true,
  package: 'egg-view-nunjucks'
};
```

```js
// config/config.default.js
// 添加 view 配置
exports.view = {
  defaultViewEngine: 'nunjucks',
  mapping: {
    '.nj': 'nunjucks',
  },
};
```

```html
<html>
  <head>
    <title>Hacker News</title>
  </head>
  <body>
    <ul class="news-view view">
      hello world
    </ul>
    <h1>hello world</h1>
  </body>
</html>
```

### 编写helper扩展

```js
// app/extend/helper.js
exports.getData = () => {
    return '我是处理后的时间'
}
```

模板引擎和ctx都可以获取helper对象

### 编写Middleware

类似koa中的中间件

```js
// app/middleware/log.js
module.exports = (options, app) => { // egg约定  1. 可以定制化配置， 2.传入 app的实例
    return async function log(ctx, next) { // 和koa写法完全一样
        console.log('我是日志！！！！！');
        await next();
    }
}
```

```js
// config/config.default.js
config.middleware = [
    'log'
];
```

1. 写一个中间件
2. 配置引入

## 渐进式开发

渐进式开发是egg里面的一种非常重要的设计思想。

1. 需要封装一些方法， 最早起的需求雏形

```js
// app/extend/context.js   给ctx对象扩展属性或者方法
module.exports = {
    get isIOS() {
        return '我不是ios'
    },
};
```

2. 插件的雏形

```
example-app
├── app
│   └── router.js
├── config
│   └── plugin.js
├── lib
│   └── plugin
│       └── egg-ua
│           ├── app
│           │   └── extend
│           │       └── context.js
│           └── package.json
├── test
│   └── index.test.js
└── package.json
```

`lib/plugin/egg-ua/package.json` 声明插件。

```json
{
  "eggPlugin": {
    "name": "ua"
  }
}
```

`config/plugin.js` 中通过 `path` 来挂载插件。

```js
// config/plugin.js
const path = require('path');
exports.ua = {
  enable: true,
  path: path.join(__dirname, '../lib/plugin/egg-ua'),
};
```

3. 抽成独立的插件, 通过npm包的形式引入

```
egg-ua
├── app
│   └── extend
│       └── context.js
├── test
│   ├── fixtures
│   │   └── test-app
│   │       ├── app
│   │       │   └── router.js
│   │       └── package.json
│   └── ua.test.js
└── package.json
```

4. 沉淀到框架

- oa-egg
- player-egg
- music-egg

### 总结

- 一般来说，当应用中有可能会复用到的代码时，直接放到 `lib/plugin` 目录去，如例子中的 `egg-ua`。
- 当该插件功能稳定后，即可独立出来作为一个 `node module` 。
- 如此以往，应用中相对复用性较强的代码都会逐渐独立为单独的插件。
- 当你的应用逐渐进化到针对某类业务场景的解决方案时，将其抽象为独立的 framework 进行发布。
- 当在新项目中抽象出的插件，下沉集成到框架后，其他项目只需要简单的重新 `npm install` 下就可以使用上，对整个团队的效率有极大的提升。

## 基础功能

### 目录结构的约定

egg的原则： 约定大于配置

```shell
egg-project
├── package.json
├── app.js (可选)
├── agent.js (可选)  // 比较独特
├── app
|   ├── router.js
│   ├── controller
│   |   └── home.js
│   ├── service (可选)
│   |   └── user.js
│   ├── middleware (可选)
│   |   └── response_time.js
│   ├── schedule (可选)
│   |   └── my_task.js
│   ├── public (可选)
│   |   └── reset.css
│   ├── view (可选)
│   |   └── home.tpl
│   └── extend (可选)
│       ├── helper.js (可选)
│       ├── request.js (可选)
│       ├── response.js (可选)
│       ├── context.js (可选)
│       ├── application.js (可选)
│       └── agent.js (可选)
├── config
|   ├── plugin.js
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test
    ├── middleware
    |   └── response_time.test.js
    └── controller
        └── home.test.js

```

#### 框架规定的目录

- `app/router.js` 用于配置 URL 路由规则。
- `app/controller/**` 用于解析用户的输入，处理后返回相应的结果。
- `app/service/**` 用于编写业务逻辑层，可选。
- `app/middleware/**` 用于编写中间件，可选。
- `app/public/**` 用于放置静态资源，可选。
- `app/extend/**` 用于框架的扩展，可选。
- `config/config.{env}.js` 用于编写配置文件。
- `config/plugin.js` 用于配置需要加载的插件。
- `test/**` 用于单元测试。
- `app.js` 和 `agent.js` 用于自定义启动时的初始化工作，可选。

#### 内置插件约定的目录

- `app/public/**` 用于放置静态资源，可选。
- `app/schedule/**` 用于定时任务，可选。

### 内置对象

在本章，我们会初步介绍一下框架中内置的一些基础对象，包括从 [Koa](http://koajs.com/) 继承而来的 4 个对象（Application, Context, Request, Response) 以及框架扩展的一些对象（Controller, Service, Helper, Config, Logger），在后续的课程中我们会经常遇到它们。

#### Application

Application 是全局应用对象，在一个应用中，只会实例化一个，它继承自 [Koa.Application](http://koajs.com/#application)，在它上面我们可以挂载一些全局的方法和对象。我们可以轻松的在插件或者应用中[扩展 Application 对象](https://eggjs.org/zh-cn/basics/extend.html#Application)。

**事件**

在框架运行时，会在 Application 实例上触发一些事件，应用开发者或者插件开发者可以监听这些事件做一些操作。作为应用开发者，我们一般会在[启动自定义脚本](https://eggjs.org/zh-cn/basics/app-start.html)中进行监听。

- `server`: 该事件一个 worker 进程只会触发一次，在 HTTP 服务完成启动后，会将 HTTP server 通过这个事件暴露出来给开发者。
- `error`: 运行时有任何的异常被 onerror 插件捕获后，都会触发 `error` 事件，将错误对象和关联的上下文（如果有）暴露给开发者，可以进行自定义的日志记录上报等处理。
- `request` 和 `response`: 应用收到请求和响应请求时，分别会触发 `request` 和 `response` 事件，并将当前请求上下文暴露出来，开发者可以监听这两个事件来进行日志记录。

```js
// app.js

module.exports = app => {
  app.once('server', server => {
    // websocket
  });
  app.on('error', (err, ctx) => {
    // report error
  });
  app.on('request', ctx => {
    // log receive request
  });
  app.on('response', ctx => {
    // ctx.starttime is set by framework
    const used = Date.now() - ctx.starttime;
    // log total cost
  });
};
```

**获取方式**

```js
// app.js
module.exports = app => {
  app.xxxx = 'xxxx';
};
```

controller文件

```js
class UserController extends Controller {
  async fetch() {
    this.ctx.body = this.app.xxxx;
  }
}
```

> 和 [Koa](http://koajs.com/) 一样，在 Context 对象上，也可以通过 `ctx.app` 访问到 Application 对象。

#### context

Context 是一个**请求级别的对象**，继承自 [Koa.Context](http://koajs.com/#context)。在每一次收到用户请求时，框架会实例化一个 Context 对象，这个对象封装了这次用户请求的信息，并提供了许多便捷的方法来获取请求参数或者设置响应信息。框架会将所有的 [Service](https://eggjs.org/zh-cn/basics/service.html) 挂载到 Context 实例上，一些插件也会将一些其他的方法和对象挂载到它上面（[egg-sequelize](https://github.com/eggjs/egg-sequelize) 会将所有的 model 挂载在 Context 上）。

**获取方式**

最常见的 Context 实例获取方式是在 [Middleware](https://eggjs.org/zh-cn/basics/middleware.html), [Controller](https://eggjs.org/zh-cn/basics/controller.html) 以及 [Service](https://eggjs.org/zh-cn/basics/service.html) 中。Controller 中的获取方式在上面的例子中已经展示过了，在 Service 中获取和 Controller 中获取的方式一样，在 Middleware 中获取 Context 实例则和 [Koa](http://koajs.com/) 框架在中间件中获取 Context 对象的方式一致。



除了在请求时可以获取 Context 实例之外， 在有些非用户请求的场景下我们需要访问 service / model 等 Context 实例上的对象，我们可以通过 `Application.createAnonymousContext()` 方法创建一个匿名 Context 实例：

```js
// app.js
module.exports = app => {
  app.beforeStart(async () => {
    const ctx = app.createAnonymousContext();
    // preload before app start
    await ctx.service.posts.load();
  });
}
```

在[定时任务](https://eggjs.org/zh-cn/basics/schedule.html)中的每一个 task 都接受一个 Context 实例作为参数，以便我们更方便的执行一些定时的业务逻辑：

```js
// app/schedule/refresh.js
exports.task = async ctx => {
  await ctx.service.posts.refresh();
};
```

####  Request & Response

Request 是一个**请求级别的对象**，继承自 [Koa.Request](http://koajs.com/#request)。封装了 Node.js 原生的 HTTP Request 对象，提供了一系列辅助方法获取 HTTP 请求常用参数。

Response 是一个**请求级别的对象**，继承自 [Koa.Response](http://koajs.com/#response)。封装了 Node.js 原生的 HTTP Response 对象，提供了一系列辅助方法设置 HTTP 响应。

```js
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const { app, ctx } = this;
    const id = ctx.request.query.id;
    ctx.response.body = app.cache.get(id);
  }
}
```

####  Controller

框架提供了一个 Controller 基类，并推荐所有的 [Controller](https://eggjs.org/zh-cn/basics/controller.html) 都继承于该基类实现。这个 Controller 基类有下列属性：

- `ctx` - 当前请求的 [Context](https://eggjs.org/zh-cn/basics/objects.html#context) 实例。
- `app` - 应用的 [Application](https://eggjs.org/zh-cn/basics/objects.html#application) 实例。
- `config` - 应用的[配置](https://eggjs.org/zh-cn/basics/config.html)。
- `service` - 应用所有的 [service](https://eggjs.org/zh-cn/basics/service.html)。
- `logger` - 为当前 controller 封装的 logger 对象。

在 Controller 文件中，可以通过两种方式来引用 Controller 基类：

```js
// app/controller/user.js

// 从 egg 上获取（推荐）
const Controller = require('egg').Controller;
class UserController extends Controller {
  // implement
}
module.exports = UserController;

// 从 app 实例上获取
module.exports = app => {
  return class UserController extends app.Controller {
    // implement
  };
};
```

#### Service

框架提供了一个 Service 基类，并推荐所有的 [Service](https://eggjs.org/zh-cn/basics/service.html) 都继承于该基类实现。

Service 基类的属性和 [Controller](https://eggjs.org/zh-cn/basics/objects.html#controller) 基类属性一致，访问方式也类似：

```js
// app/service/user.js

// 从 egg 上获取（推荐）
const Service = require('egg').Service;
class UserService extends Service {
  // implement
}
module.exports = UserService;

// 从 app 实例上获取
module.exports = app => {
  return class UserService extends app.Service {
    // implement
  };
};
```

#### Helper

Helper 用来提供一些实用的 utility 函数。它的作用在于我们可以将一些常用的动作抽离在 helper.js 里面成为一个独立的函数，这样可以用 JavaScript 来写复杂的逻辑，避免逻辑分散各处，同时可以更好的编写测试用例。

Helper 自身是一个类，有和 [Controller](https://eggjs.org/zh-cn/basics/objects.html#controller) 基类一样的属性，它也会在每次请求时进行实例化，因此 Helper 上的所有函数也能获取到当前请求相关的上下文信息。

**获取方式**

可以在 Context 的实例上获取到当前请求的 Helper(`ctx.helper`) 实例。

```js
// app/controller/user.js
class UserController extends Controller {
  async fetch() {
    const { app, ctx } = this;
    const id = ctx.query.id;
    const user = app.cache.get(id);
    ctx.body = ctx.helper.formatUser(user);
  }
}
```

除此之外，Helper 的实例还可以在模板中获取到。

#### config

我们可以通过 `app.config` 从 Application 实例上获取到 config 对象，也可以在 Controller, Service, Helper 的实例上通过 `this.config` 获取到 config 对象。



### 运行环境

`EGG_SERVER_ENV=prod npm start`

**获取运行环境:**

框架提供了变量 `app.config.env` 来表示应用当前的运行环境。

> 很多 Node.js 应用会使用 `NODE_ENV` 来区分运行环境，但 `EGG_SERVER_ENV` 区分得更加精细。一般的项目开发流程包括本地开发环境、测试环境、生产环境等

框架默认支持的运行环境及映射关系（如果未指定 `EGG_SERVER_ENV` 会根据 `NODE_ENV` 来匹配）

![image-20190629122051604](E:/Front End/0进阶课程/阶段4：Node.js进阶/4-6 Web应用开发框架/assets/image-20190629122051604.png)

例如，当 `NODE_ENV` 为 `production` 而 `EGG_SERVER_ENV` 未指定时，框架会将 `EGG_SERVER_ENV` 设置成 `prod`。

> 比如，要为开发流程增加集成测试环境 SIT。将 `EGG_SERVER_ENV` 设置成 `sit`（并建议设置 `NODE_ENV = production`），启动时会加载 `config/config.sit.js`，运行环境变量 `app.config.env` 会被设置成 `sit`。

### Config配置

框架提供了强大且可扩展的配置功能，可以自动合并应用、插件、框架的配置，按顺序覆盖，且可以根据环境维护不同的配置。合并后的配置可直接从 `app.config` 获取。

#### 多环境配置

框架支持根据环境来加载配置，定义多个环境的配置文件。

```
config
|- config.default.js
|- config.prod.js
|- config.unittest.js
`- config.local.js
```

`config.default.js` 为默认的配置文件，所有环境都会加载这个配置文件，一般也会作为开发环境的默认配置文件。

当指定 env 时会同时加载对应的配置文件，并覆盖默认配置文件的同名配置。如 `prod` 环境会加载 `config.prod.js` 和 `config.default.js` 文件，`config.prod.js` 会覆盖 `config.default.js` 的同名配置。

#### 配置写法

配置文件返回的是一个 object 对象，可以覆盖框架的一些配置，应用也可以将自己业务的配置放到这里方便管理。

```js
// 配置 logger 文件的目录，logger 默认配置由框架提供
module.exports = {
  logger: {
    dir: '/home/admin/logs/demoapp',
  },
};
```

配置文件也可以简化的写成 `exports.key = value` 形式

```js
exports.keys = 'my-cookie-secret-key';
exports.logger = {
  level: 'DEBUG',
};

```

配置文件也可以返回一个 function，可以接受 appInfo 参数

```js
// 将 logger 目录放到代码目录下
const path = require('path');
module.exports = appInfo => {
  return {
    logger: {
      dir: path.join(appInfo.baseDir, 'logs'),
    },
  };
};
```

内置的 appInfo 有:

![image-20190629124037397](E:/Front End/0进阶课程/阶段4：Node.js进阶/4-6 Web应用开发框架/assets/image-20190629124037397.png)

`appInfo.root` 是一个优雅的适配，比如在服务器环境我们会使用 `/home/admin/logs` 作为日志目录，而本地开发时又不想污染用户目录，这样的适配就很好解决这个问题。

#### 配置加载顺序

应用、插件、框架都可以定义这些配置，而且目录结构都是一致的，但存在优先级（**应用 > 框架 > 插件**），相对于此运行环境的优先级会更高。

比如在 prod 环境加载一个配置的加载顺序如下，后加载的会覆盖前面的同名配置。

```js
-> 插件 config.default.js
-> 框架 config.default.js
-> 应用 config.default.js
-> 插件 config.prod.js
-> 框架 config.prod.js
-> 应用 config.prod.js
```

#### 合并规则

```js
const a = {
  arr: [ 1, 2 ],
};
const b = {
  arr: [ 3 ],
};
extend(true, a, b);

// [3]
```

#### 配置结果

框架在启动时会把合并后的最终配置 dump 到 `run/application_config.json`（worker 进程）和 `run/agent_config.json`（agent 进程）中，可以用来分析问题。

配置文件中会隐藏一些字段，主要包括两类:

- 如密码、密钥等安全字段。
- 如函数、Buffer 等类型，`JSON.stringify` 后的内容特别大

还会生成 `run/application_config_meta.json`（worker 进程）和 `run/agent_config_meta.json`（agent 进程）文件，用来排查属性的来源，如：

```json
{
  "logger": {
    "dir": "/path/to/config/config.default.js"
  }
}
```

### 中间件

我们介绍了 Egg 是基于 Koa 实现的，所以 Egg 的中间件形式和 Koa 的中间件形式是一样的，都是基于[洋葱圈模型](https://eggjs.org/zh-cn/intro/egg-and-koa.html#midlleware)。每次我们编写一个中间件，就相当于在洋葱外面包了一层。

#### 编写中间件

```js
// app/middleware/gzip.js
const isJSON = require('koa-is-json');
const zlib = require('zlib');

async function gzip(ctx, next) {
  await next();  // app.use过之后的下一个函数

  // 后续中间件执行完成后将响应体转换成 gzip
  let body = ctx.body;
  if (!body) return;
  if (isJSON(body)) body = JSON.stringify(body);

  // 设置 gzip body，修正响应头
  const stream = zlib.createGzip();
  stream.end(body);
  ctx.body = stream;
  ctx.set('Content-Encoding', 'gzip');
}
```

#### 配置

一般来说中间件也会有自己的配置。在框架中，一个完整的中间件是包含了配置处理的。我们约定一个中间件是一个放置在 `app/middleware` 目录下的单独文件，它需要 exports 一个普通的 function，接受两个参数：

- options: 中间件的配置项，框架会将 `app.config[${middlewareName}]` 传递进来。
- app: 当前应用 Application 的实例。

我们将上面的 gzip 中间件做一个简单的优化，让它支持指定只有当 body 大于配置的 threshold 时才进行 gzip 压缩，我们要在 `app/middleware` 目录下新建一个文件 `gzip.js`

```js
// app/middleware/gzip.js
const isJSON = require('koa-is-json');
const zlib = require('zlib');

module.exports = options => {
  return async function gzip(ctx, next) {
    await next();

    // 后续中间件执行完成后将响应体转换成 gzip
    let body = ctx.body;
    if (!body) return;

    // 支持 options.threshold
    if (options.threshold && ctx.length < options.threshold) return;

    if (isJSON(body)) body = JSON.stringify(body);

    // 设置 gzip body，修正响应头
    const stream = zlib.createGzip();
    stream.end(body);
    ctx.body = stream;
    ctx.set('Content-Encoding', 'gzip');
  };
};
```

#### 使用中间件

中间件编写完成后，我们还需要手动挂载，支持以下方式：

```js
module.exports = {
  // 配置需要的中间件，数组顺序即为中间件的加载顺序
  middleware: [ 'gzip' ],

  // 配置 gzip 中间件的配置
  gzip: {
    threshold: 1024, // 小于 1k 的响应体不压缩
  },
};
```

#### 在框架和插件中使用中间件

可以通过app的config对象去个框架添加中间件，因为可能中间件执行顺序有一些要求。

```js
// app.js
module.exports = app => {
  // 在中间件最前面统计请求时间
  app.config.coreMiddleware.unshift('report');
};

// app/middleware/report.js
module.exports = () => {
  return async function (ctx, next) {
    const startTime = Date.now();
    await next();
    // 上报请求时间
    reportTime(Date.now() - startTime);
  }
};
```

应用层定义的中间件（`app.config.appMiddleware`）和框架默认中间件（`app.config.coreMiddleware`）都会被加载器加载，并挂载到 `app.middleware` 上。

#### 单个路由生效

中间件对象也挂载在app下面

```js
module.exports = app => {
  const gzip = app.middleware.gzip({ threshold: 1024 });
  app.router.get('/needgzip', gzip, app.controller.handler);
};
```

#### 框架默认中间件

除了应用层加载中间件之外，框架自身和其他的插件也会加载许多中间件。所有的这些自带中间件的配置项都通过在配置中修改中间件同名配置项进行修改，例如[框架自带的中间件](https://github.com/eggjs/egg/tree/master/app/middleware)中有一个 bodyParser 中间件（框架的加载器会将文件名中的各种分隔符都修改成驼峰形式的变量名），我们想要修改 bodyParser 的配置，只需要在 `config/config.default.js` 中编写

```js
module.exports = {
  bodyParser: {
    jsonLimit: '10mb',
  },
};
```

#### 使用koa的中间件

以 [koa-compress](https://github.com/koajs/compress) 为例，在 Koa 中使用时：

```js
const koa = require('koa');
const compress = require('koa-compress');

const app = koa();

const options = { threshold: 2048 };
app.use(compress(options));
```

我们按照框架的规范来在应用中加载这个 Koa 的中间件：

```js
// app/middleware/compress.js
// koa-compress 暴露的接口(`(options) => middleware`)和框架对中间件要求一致
module.exports = require('koa-compress');
```

```
// config/config.default.js
module.exports = {
  middleware: [ 'compress' ],
  compress: {
    threshold: 2048,
  },
};
```

如果使用到的 Koa 中间件不符合入参规范，则可以自行处理下：

```js
// config/config.default.js
module.exports = {
  webpack: {
    compiler: {},
    others: {},
  },
};

// app/middleware/webpack.js
const webpackMiddleware = require('some-koa-middleware');

module.exports = (options, app) => {
  return webpackMiddleware(options.compiler, options.others);
}
```

#### 通用配置

无论是应用层加载的中间件还是框架自带中间件，都支持几个通用的配置项：

- enable：控制中间件是否开启。
- match：设置只有符合某些规则的请求才会经过这个中间件。
- ignore：设置符合某些规则的请求不经过这个中间件。

如果我们的应用并不需要默认的 bodyParser 中间件来进行请求体的解析，此时我们可以通过配置 enable 为 false 来关闭它

```js
module.exports = {
  bodyParser: {
    enable: false,
  },
};
```

如果我们想让 gzip 只针对 `/static` 前缀开头的 url 请求开启，我们可以配置 match 选项

```js
module.exports = {
  gzip: {
    match: '/static',
  },
};
```

match 和 ignore 支持多种类型的配置方式

1. 字符串：当参数为字符串类型时，配置的是一个 url 的路径前缀，所有以配置的字符串作为前缀的 url 都会匹配上。 当然，你也可以直接使用字符串数组。
2. 正则：当参数为正则时，直接匹配满足正则验证的 url 的路径。
3. 函数：当参数为一个函数时，会将请求上下文传递给这个函数，最终取函数返回的结果（true/false）来判断是否匹配。

```js
module.exports = {
  gzip: {
    match(ctx) {
      // 只有 ios 设备才开启
      const reg = /iphone|ipad|ipod/i;
      return reg.test(ctx.get('user-agent'));
    },
  },
};
```

### 路由

Router 主要用来描述请求 URL 和具体承担执行动作的 Controller 的对应关系， 框架约定了 `app/router.js` 文件用于统一所有路由规则。

通过统一的配置，我们可以避免路由规则逻辑散落在多个地方，从而出现未知的冲突，集中在一起我们可以更方便的来查看全局的路由规则。

#### 如何定义Router

- `app/router.js` 里面定义 URL 路由规则

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.get('/user/:id', controller.user.info);
};
```

- `app/controller` 目录下面实现 Controller

```js
// app/controller/user.js
class UserController extends Controller {
  async info() {
    const { ctx } = this;
    ctx.body = {
      name: `hello ${ctx.params.id}`,
    };
  }
}
```

支持 get，post 等所有 HTTP 方法

- router.get - GET
- router.put - PUT
- router.post - POST
- router.patch - PATCH
- router.delete - DELETE

#### restful风格的URL定义

[http://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html](http://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html)

如果想通过 RESTful 的方式来定义路由， 我们提供了 `app.resources('routerName', 'pathMatch', controller)` 快速在一个路径上生成 [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) 路由结构。

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.resources('/api/user', controller.posts);
};
```

上面代码就在 `/posts` 路径上部署了一组 CRUD 路径结构，对应的 Controller 为 `app/controller/posts.js` 接下来， 你只需要在 `posts.js` 里面实现对应的函数就可以了。

| 请求方式 | 路径         | 路径名 | 对应控制器的方法          | 说明             |
| -------- | ------------ | ------ | ------------------------- | ---------------- |
| GET      | /aa          |        | app.controller.aa.index   | 获取所有信息     |
| GET      | /aa/new      |        | app.controller.aa.new     |                  |
| GET      | /aa/:id      |        | app.controller.aa.show    | 查找指定的数据   |
| GET      | /aa/:id/edit |        | app.controller.aa.edit    | 指定数据进行更改 |
| POST     | /aa          |        | app.controller.aa.create  | 创建新增数据     |
| PUT      | /aa/:id      |        | app.controller.aa.update  | 更新指定数据     |
| DELETE   | /aa/:id      |        | app.controller.aa.destroy | 删除指定数据     |

#### 获取参数

```js
// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
};

// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`;
};
```

```js
// app/router.js
module.exports = app => {
  app.router.get('/user/:id/:name', app.controller.user.info);
};

// app/controller/user.js
exports.info = async ctx => {
  ctx.body = `user: ${ctx.params.id}, ${ctx.params.name}`;
};

// curl http://127.0.0.1:7001/user/123/xiaoming
```

### 控制器

简单的说 Controller 负责**解析用户的输入，处理后返回相应的结果**

#### 作用

- 在 [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) 接口中，Controller 接受用户的参数，从数据库中查找内容返回给用户或者将用户的请求更新到数据库中。
  - 返回json
- 在 HTML 页面请求中，Controller 根据用户访问不同的 URL，渲染不同的模板得到 HTML 返回给用户。
- 在代理服务器中，Controller 将用户的请求转发到其他服务器上，并将其他服务器的处理结果返回给用户。
  - 调用第三方的接口

#### 推荐用法(适合在controller中完成的逻辑)

1. 获取用户通过 HTTP 传递过来的请求参数。(解析参数)
2. 校验、组装参数。(校验)
3. 调用 Service 进行业务处理，必要时处理转换 Service 的返回结果，让它适应用户的需求。(数据库查询)
4. 通过 HTTP 将结果响应给用户。(返回response)

#### 如何编写controller

```js
'use strict';
// import HttpController from './base/http';
const Controller = require('egg').Controller;

class HomeController extends Controller {
  async index() {
    const { ctx } = this;
    ctx.body = await this.service.user.find() // ctx 来自koa
  }

  // 1. 需要在 controller定义怎么渲染
  async hello() {
    const { ctx } = this;
    await ctx.render('hello.nj'); // 自动的读取view/hello.nj
  }

  async addUser() {
    const { ctx } = this;
    console.log(ctx.request.body);
    console.log(ctx.csrf)
    ctx.body = 'success';
  }

  async redirect() {
    const { ctx } = this;
    ctx.redirect('https://taobao.com');
  }
}

module.exports = HomeController;

```

#### router调用

```js
// app/router.js
module.exports = app => {
  const { router, controller } = app;
  router.post('/api/posts', controller.post.create);
}
```

**Controller 支持多级目录**，例如如果我们将上面的 Controller 代码放到 `app/controller/sub/post.js` 中，则可以在 router 中这样使用：

```js
// app/router.js
module.exports = app => {
  app.router.post('/api/posts', app.controller.sub.post.create);
}
```

#### Controller的属性

项目中的 Controller 类继承于 `egg.Controller`，会有下面几个属性挂在 `this` 上。

- `this.ctx`: 当前请求的上下文 [Context](https://eggjs.org/zh-cn/basics/extend.html#context) 对象的实例，通过它我们可以拿到框架封装好的处理当前请求的各种便捷属性和方法。
- `this.app`: 当前应用 [Application](https://eggjs.org/zh-cn/basics/extend.html#application) 对象的实例，通过它我们可以拿到框架提供的全局对象和方法。
- `this.service`：应用定义的 [Service](https://eggjs.org/zh-cn/basics/service.html)，通过它我们可以访问到抽象出的业务层，等价于 `this.ctx.service` 。
- `this.config`：应用运行时的[配置项](https://eggjs.org/zh-cn/basics/config.html)。
- `this.logger`：logger 对象，上面有四个方法（`debug`，`info`，`warn`，`error`），分别代表打印四个不同级别的日志，使用方法和效果与 [context logger](https://eggjs.org/zh-cn/core/logger.html#context-logger) 中介绍的一样，但是通过这个 logger 对象记录的日志，在日志前面会加上打印该日志的文件路径，以便快速定位日志打印位置。

#### 自定义Controller基类

按照类的方式编写 Controller，不仅可以让我们更好的对 Controller 层代码进行抽象（例如将一些统一的处理抽象成一些私有方法），还可以通过自定义 Controller 基类的方式封装应用中常用的方法。

```js
// app/controller/base/http.js

const Controller = require('egg').Controller;

class HttpController extends Controller {
  success(data) {
    this.ctx.body = {
        msg: 'success',
        code: 0,
        data
    }
  }
}

module.exports = HttpController;

```

```js
//app/controller/post.js
const Controller = require('../core/base_controller');
class PostController extends Controller {
  async list() {
    const posts = await this.service.listByUser(this.user);
    this.success(posts);
  }
}
```

#### 获取request参数

- query   地址栏参数， 会丢弃重复的参数
- params   route参数
- **queries 地址栏参数，不会丢弃重复的参数**
- **request.body的参数**   框架内置了 [bodyParser](https://github.com/koajs/bodyparser) 中间件来对这两类格式的请求 body 解析成 object 挂载到 `ctx.request.body` 上

> 一个常见的错误是把 ctx.request.body 和 ctx.body 混淆，后者其实是 ctx.response.body 的简写。**

#### csrf防范

egg会默认带上csrf防护，不加上token是取不到post中的参数的。

- 从ctx.csrf中读取token

- 通过header的x-csrf-token字段携带过来

#### 重定向

框架通过 security 插件覆盖了 koa 原生的 `ctx.redirect` 实现，以提供更加安全的重定向。

- `ctx.redirect(url)` 如果不在配置的白名单域名内，则禁止跳转。
- `ctx.unsafeRedirect(url)` 不判断域名，直接跳转，一般不建议使用，明确了解可能带来的风险后使用。

```js
// config/config.default.js
exports.security = {
  domainWhiteList:['.domain.com'],  // 安全白名单，以 . 开头
};
```

### 服务(Service)

简单来说，Service 就是专门请求和组装数据的。

####  使用 场景

- 复杂数据的处理，比如要展现的信息需要从数据库获取，还要经过一定的规则计算，才能返回用户显示。或者计算完成后，更新到数据库。
- 第三方服务的调用，比如 GitHub 信息获取等。

```js
// app/service/user
const Service = require('egg').Service;

class UserService extends Service {
    async find(uid) {
        return [1, 2, 3, 4];
    }
}

module.exports = UserService;
```

```js
'use strict';
// import HttpController from './base/http';
const Controller = require('egg').Controller;

class HomeController extends Controller {
  async index() {
    const { ctx } = this;
    ctx.body = await this.service.user.find() // ctx 来自koa
  }
}

module.exports = HomeController;

```

### 插件

插件机制是我们框架的一大特色。它不但可以保证框架核心的足够精简、稳定、高效，还可以促进业务逻辑的复用，生态圈的形成。

- Koa 已经有了中间件的机制，为啥还要插件呢？
- 中间件、插件、应用它们之间是什么关系，有什么区别？

#### 为什么要插件

**使用 Koa 中间件过程中发现了下面一些问题：**

1. 中间件加载其实是有先后顺序的，但是中间件自身却无法管理这种顺序，只能交给使用者。这样其实非常不友好，一旦顺序不对，结果可能有天壤之别。
2. 中间件的定位是拦截用户请求，并在它前后做一些事情，例如：鉴权、安全检查、访问日志等等。但实际情况是，有些功能是和请求无关的，例如：定时任务、消息订阅、后台逻辑等等。
3. 有些功能包含非常复杂的初始化逻辑，需要在应用启动的时候完成。这显然也不适合放到中间件中去实现。

#### 中间件、插件、应用的关系

一个插件其实就是一个『迷你的应用』，和应用（app）几乎一样：

- 它包含了 Service、中间件、配置、框架扩展等等。
- 它没有独立的 Router 和 Controller。(插件一般不写业务逻辑)
- 它没有 plugin.js，只能声明跟其他插件的依赖，而不能决定其他插件的开启与否。

#### 使用插件

插件一般通过 npm 模块的方式进行复用：

```shell
npm i egg-mysql --save
```

然后需要在应用或框架的 `config/plugin.js` 中声明：

```js
// config/plugin.js
// 使用 mysql 插件
exports.mysql = {
  enable: true,
  package: 'egg-mysql',
};
```

#### 根据环境配置

同时，我们还支持 `plugin.{env}.js` 这种模式，会根据[运行环境](https://eggjs.org/zh-cn/basics/env.html)加载插件配置。

```js
// config/plugin.local.js
exports.dev = {
  enable: true,
  package: 'egg-dev',
};
```

####  引入 

- `package` 是 `npm` 方式引入，也是最常见的引入方式
- `path` 是绝对路径引入，如应用内部抽了一个插件，但还没达到开源发布独立 `npm` 的阶段，或者是应用自己覆盖了框架的一些插件

```js
// config/plugin.js
const path = require('path');
exports.mysql = {
  enable: true,
  path: path.join(__dirname, '../lib/plugin/egg-mysql'),
};
```

#### 如何写一个插件

你可以直接使用 [egg-boilerplate-plugin](https://github.com/eggjs/egg-boilerplate-plugin) 脚手架来快速上手。

```
$ mkdir egg-hello && cd egg-hello
$ npm init egg --type=plugin
$ npm i
```

一个插件其实就是一个『迷你的应用』，下面展示的是一个插件的目录结构，和应用（app）几乎一样。

```js
. egg-hello
├── package.json
├── app.js (可选)
├── agent.js (可选)
├── app
│   ├── extend (可选)
│   |   ├── helper.js (可选)
│   |   ├── request.js (可选)
│   |   ├── response.js (可选)
│   |   ├── context.js (可选)
│   |   ├── application.js (可选)
│   |   └── agent.js (可选)
│   ├── service (可选)
│   └── middleware (可选)
│       └── mw.js
├── config
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test
    └── middleware
        └── mw.test.js
```

1. 插件没有独立的 router 和 controller。这主要出于几点考虑：

- 路由一般和应用强绑定的，不具备通用性。
- 一个应用可能依赖很多个插件，如果插件支持路由可能导致路由冲突。
- 如果确实有统一路由的需求，可以考虑在插件里通过中间件来实现。

2. 插件需要在 `package.json` 中的 `eggPlugin` 节点指定插件特有的信息：

- `{String} name` - 插件名（必须配置），具有唯一性，配置依赖关系时会指定依赖插件的 name。
- `{Array} dependencies` - 当前插件强依赖的插件列表（如果依赖的插件没找到，应用启动失败）。
- `{Array} optionalDependencies` - 当前插件的可选依赖插件列表（如果依赖的插件未开启，只会 warning，不会影响应用启动）。
- `{Array} env` - 只有在指定运行环境才能开启，具体有哪些环境可以参考[运行环境](https://eggjs.org/zh-cn/basics/env.html)。此配置是可选的，一般情况下都不需要配置。

```json
{
  "name": "egg-rpc",
  "eggPlugin": {
    "name": "rpc",
    "dependencies": [ "registry" ],
    "optionalDependencies": [ "vip" ],
    "env": [ "local", "test", "unittest", "prod" ]
  }
}
```

#### 插件能做什么？

- 扩展内置对象的接口
  - `app/extend/request.js` - 扩展 Koa#Request 类
  - `app/extend/response.js` - 扩展 Koa#Response 类
  - `app/extend/context.js` - 扩展 Koa#Context 类
  - `app/extend/helper.js` - 扩展 Helper 类
  - `app/extend/application.js` - 扩展 Application 类
  - `app/extend/agent.js` - 扩展 Agent 类
- 插入自定义中间件
- 在应用启动时做一些初始化工作
- 设置定时任务

### 定时任务

会有许多场景需要执行一些定时任务，例如：

1. 定时上报应用状态。
2. 定时从远程接口更新本地缓存。
3. 定时进行文件切割、临时文件删除。

**框架提供了一套机制来让定时任务的编写和维护更加优雅。**

#### 编写定时任务

所有的定时任务都统一存放在 `app/schedule` 目录下，每一个文件都是一个独立的定时任务，可以配置定时任务的属性和要执行的方法。

```js
const Subscription = require('egg').Subscription;

class LogSubscription extends Subscription {
  // 通过 schedule 属性来设置定时任务的执行间隔等配置
  static get schedule() {
    return {
      interval: '1s', // 1 分钟间隔
      type: 'worker', // 指定所有的 worker 都需要执行
    };
  }

  // subscribe 是真正定时任务执行时被运行的函数
  async subscribe() {
    console.log('我是定时任务')
  }
}

module.exports = LogSubscription;
```

另一种写法：

```js
module.exports = {
  schedule: {
    interval: '1s', // 1 分钟间隔
    type: 'all', // 指定所有的 worker 都需要执行
  },
  async task(ctx) {
    console.log('我是定时任务')
  },
};
```

#### 参数

- interval
  - 数字类型，单位为毫秒数，例如 `5000`。
  - 字符类型，会通过 [ms](https://github.com/zeit/ms) 转换成毫秒数，例如 `5s`。
- type
  - `worker` 类型：每台机器上只有一个 worker 会执行这个定时任务，每次执行定时任务的 worker 的选择是随机的。
  - `all` 类型：每台机器上的每个 worker 都会执行这个定时任务。



### 自定义启动 

我们常常需要在应用启动期间进行一些初始化工作，等初始化完成后应用才可以启动成功，并开始对外提供服务。

框架提供了统一的入口文件（`app.js`）进行启动过程自定义，这个文件返回一个 Boot 类，我们可以通过定义 Boot 类中的生命周期方法来执行启动应用过程中的初始化工作。

框架提供了这些 [生命周期函数](https://eggjs.org/zh-cn/advanced/loader.html#life-cycles)供开发人员处理：

- 配置文件即将加载，这是最后动态修改配置的时机（`configWillLoad`）
- 配置文件加载完成（`configDidLoad`）
- 文件加载完成（`didLoad`）
- 插件启动完毕（`willReady`）
- worker 准备就绪（`didReady`）
- 应用启动完成（`serverDidReady`）
- 应用即将关闭（`beforeClose`）

```js
// app.js
class AppBootHook {
    constructor(app) {
      this.app = app;
    }
  
    configWillLoad() {
         // 此时 config 文件已经被读取并合并，但是还并未生效
         // 这是应用层修改配置的最后时机
         // 注意：此函数只支持同步调用
        console.log('configWillLoad')
     
  
      // 例如：参数中的密码是加密的，在此处进行解密
    //   this.app.config.mysql.password = decrypt(this.app.config.mysql.password);
    //   // 例如：插入一个中间件到框架的 coreMiddleware 之间
    //   const statusIdx = this.app.config.coreMiddleware.indexOf('status');
    //   this.app.config.coreMiddleware.splice(statusIdx + 1, 0, 'limit');
    }
  
    async didLoad() {
        console.log('didLoad')
      // 所有的配置已经加载完毕
      // 可以用来加载应用自定义的文件，启动自定义的服务
  
      // 例如：创建自定义应用的示例
    //   this.app.queue = new Queue(this.app.config.queue);
    //   await this.app.queue.init();
  
    //   // 例如：加载自定义的目录
    //   this.app.loader.loadToContext(path.join(__dirname, 'app/tasks'), 'tasks', {
    //     fieldClass: 'tasksClasses',
    //   });
    }
  
    async willReady() {
        console.log('willReady')
      // 所有的插件都已启动完毕，但是应用整体还未 ready
      // 可以做一些数据初始化等操作，这些操作成功才会启动应用
  
      // 例如：从数据库加载数据到内存缓存
    //   this.app.cacheData = await this.app.model.query(QUERY_CACHE_SQL);
    }
  
    async didReady() {
        console.log('didReady')
      // 应用已经启动完毕
  
    //   const ctx = await this.app.createAnonymousContext();
    //   await ctx.service.Biz.request();
    }
  
    async serverDidReady() {
        console.log('serverDidReady')
      // http / https server 已启动，开始接受外部请求
      // 此时可以从 app.server 拿到 server 的实例
  
    //   this.app.server.on('timeout', socket => {
    //     // handle socket timeout
    //   });
    }
  }
  
  module.exports = AppBootHook;
```

**注意：在自定义生命周期函数中不建议做太耗时的操作，框架会有启动的超时检测。**