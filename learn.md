# koa 源码解读

我觉得比较简单粗暴的去分析源码的方式就是分析一下一段很常用的代码究竟是怎么运行的吧。可以先来看入门级代码。

```js
const koa = require("koa");
const app = new koa();

app.use(async (ctx, next) => {
  ctx.body = "1";
  next();
  ctx.body += "2";
});

app.use(async (ctx, next) => {
  ctx.body += "3";
  next();
  ctx.body += "4";
});
app.use(async (ctx, next) => {
  ctx.body += "5";
  next();
  ctx.body += "6";
});

app.context.title = "hello world";

app.listen(3000);
```

## Application 部分

这一部分是 koa 的入口文件

### use

只是在中间件里面加入了要执行的函数，见代码

```js
use(fn) {
  if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
  if (isGeneratorFunction(fn)) {
    deprecate('Support for generators will be removed in v3. ' +
              'See the documentation for examples of how to convert old middleware ' +
              'https://github.com/koajs/koa/blob/master/docs/migration.md');
    fn = convert(fn);
  }
  debug('use %s', fn._name || fn.name || '-');
  this.middleware.push(fn);
  return this;
}
```

所以前面一堆都只是在 middleware 数组里面存储了待处理的函数，在这个 middleware 数组内的函数都形如:

```js
async (ctx, next) => {
  next();
};
```

他们执行的时机在这个比较不起眼的监听函数

```js
listen(...args) {
  debug('listen');
  const server = http.createServer(this.callback());
  return server.listen(...args);
}
```

其实也就是直接调用了类里面的 callback

```js
callback() {
  // 这个compose是灵魂之笔，可以后续进行分析，现在可以简单理解成fn是当前middleware数组里面函数依次串行执行的触发器
  const fn = compose(this.middleware);

  if (!this.listenerCount('error')) this.on('error', this.onerror);

  const handleRequest = (req, res) => {
    const ctx = this.createContext(req, res);
    return this.handleRequest(ctx, fn);
  };

  return handleRequest;
}
```

这么看 callback 是所有逻辑开始运行的起源，里面做了三件事情

1、通过 compose 把所有 middlewares 里面的函数连接在一起，生成一个名为 fn 的串行执行触发器

2、通过现在的 req 和 res 生成上下文

3、通过 handleRequest 方法在 ctx 上下文中去执行这个 fn

因此接下来会围绕两个问题展开思考

1、compose 是怎么把 middlewares 里面的函数都连接在一起的

2、生成 context 上下文的过程是怎么样的

### compose

就是这个函数把之前我们 use 里面加入的函数串行起来，结合以上的实例，可以看下他的执行流程。

分析下代码

```js
function compose(middleware) {
  if (!Array.isArray(middleware))
    throw new TypeError("Middleware stack must be an array!");
  for (const fn of middleware) {
    if (typeof fn !== "function")
      throw new TypeError("Middleware must be composed of functions!");
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // 这里的index和i可以看作双指针，i指向当前执行的中间件，index指向上次执行的中间件
    let index = -1;
    // 递归开始
    return dispatch(0);
    function dispatch(i) {
      if (i <= index)
        return Promise.reject(new Error("next() called multiple times"));
      index = i;
      let fn = middleware[i];
      // 如果发现已经执行完所有中间件，就把fn设为next，而在代码的运行过程中，next并没有赋值，所以下面会走到 `Promise.resolve()` 也就是最里面的函数结束后层层从递归里面出来（洋葱模型的后半段）
      if (i === middleware.length) fn = next;
      if (!fn) return Promise.resolve();
      try {
        // 每次next()的时候其实就相当于运行指向下一个中间件的dispatch函数进行递归（洋葱模型的前半段）
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err);
      }
    }
  };
}
```

**createContext** 相当于是剩下三个库的入口，核心功能就是生成上下文

emmm，需要先注意下[Object.create() 和 new 的区别](https://www.jianshu.com/p/358d04e054b2)

在 koa 实例初始化的时候，有三个变量也跟着初始化生成了

```js
this.context = Object.create(context);
this.request = Object.create(request);
this.response = Object.create(response);
```

## context + request + response 部分

emmm,感觉就是一个大对象，通过 delegate 挂载了一些方法和变量.之后三者对 app、request、response 等变量进行共享

```js
context.app = request.app = response.app = this;
context.req = request.req = response.req = req;
context.res = request.res = response.res = res;
request.ctx = response.ctx = context;
request.response = response;
response.request = request;
context.originalUrl = request.originalUrl = req.url;
context.state = {};
```

这里要注意一个问题，看过代码可以发现，其实 request 和 response 只是对原生 req 和 res 的加工。

那么要清楚我们原生的 req 和 res 是从哪里来的呢？ -> 其实就是在,没错又是上文提到的那个不起眼的监听函数

```js
listen(...args) {
  debug('listen');
  const server = http.createServer(this.callback());
  return server.listen(...args);
}
```

起源就在于这个 callback 函数，表面上看他什么都没传，但实际上我们可以写一个简化版本去描述这个流程

```js
const http = require("http");

const callback = () => {
  const handleRequet = (req, res) => {
    console.log("req", req);
    res.write("10086") && res.end();
  };
  return handleRequet;
};

http.createServer(callback()).listen(3000);
```

没错，一目了然，就是在监听的时候 koa 拿到了原始的 req 和 res。

## 总结

最后用十级大白话水平说下对 koa 的理解吧。

1、koa 实例化时，只是生成了一个具备存储功能的实例，里面核心的内容有 middleware、context 实例、request 实例、response 实例。

2、context 实例可以看作是 koa 的内容容器，里面主要是集成了 request 实例和 response 实例(其实也就是原生 req 和 res 的加强版)和一些错误处理。

3、middleware 可以看作是 koa 的逻辑处理器，里面主要就是对 req、res 层层加工，通过 app.use 注入

4、app.listen 开始监听，路人甲向 koa 发起请求，koa 为这次请求创建了一个 context 上下文实例，并对原生 req 和 res 进行加工得到了加强版的 requset、response。紧接着这次请求开始进入洋葱模型，依次串联执行中间件们的逻辑进行层层加工，最后在处理后的此次请求专属上下文里，为本次访问提供服务。


# 其他

## koa和express的比较

express确实用的不多，只有个人项目简单写过，这里简单过下官方的文档吧，看完这个文档总结出自己的心得

1、koa并不是用来代替express的，express还有更适合自己用的地方，他优秀在自己便集成了很多功能比如Routing、Templating	、Sending Files、JSONP，大而全而且更贴近于node原生的写法
2、koa的优势在于：语法相对现代，基于promise的控制流，避免了callback hell。同时相对轻量，可以选择自己需要的来自第三方的中间件进行组装，模块思维更强。
3、个人的想法在现在的时代里，如果追求大而全的node后端框架, 完全可以用egg(当然个人还是更挺Nest.js或者Midway的).如果想追求灵活性强一点的，个性化一点的可以试试koa.那express适合来干什么呢？如果工作内容不用维护较老的工程的话, express用来做node的学习工具尚可, 毕竟人家官方都不打算出Express4.0惹


## koa是如何完成单元测试的

测试驱动开发: 每个模块，在完成大体功能需要进行检验和测试之前，先完成单元测试的测试用例的编写，接下来的过程其实就是修改代码直至通过所有用例的过程。

egg-bin 内置了Mocha、co-mocha、power-assert，nyc 等模块，koa直接通过这个库对各个功能进行测试。

test目录下大体过了一遍，感觉单元测试写的很清晰，因为之前没有写过单测所以，如果以后需要写单测的话，koa是个不错的借鉴。