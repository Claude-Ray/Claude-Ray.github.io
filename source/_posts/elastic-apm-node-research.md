---
title: Node.js APM 产品调研：Elastic APM
date: 2019-05-27 22:46:05
tags: [Node.js,APM,Elastic-APM,V8]
categories: Node.js
---

# 前言

根据上一篇《[Node.js APM 产品调研](https://claude-ray.github.io/2019/05/19/node-apm-product-research/)》的市场调研结果，笔者更青睐 Elastic APM 这个开源产品，故决定带来它的一篇专题介绍。

尽管团队已经开始试用，但踩坑时间较短，与其编写测评，不如先带大家走进这个项目，剖析个别令人感兴趣的技术点。

<!--more-->

# 基础介绍
## 项目背景
从 github 的信息来看，项目从 2011 年 11 月开工，已经不算新项目，期间基本就是单人维护的状态，进展到现在颇为不易。

两任作者分别是 Sentry 的核心成员 Matt Robenolt，以及 Elastic 团队的 Node.js 专职开发 Thomas Watson，同时他也是 Node.js 团队的核心贡献者之一。

对 Elastic APM 完全没有接触过的读者，可以先阅读 nswbmw/node-in-debugging 中的[介绍](https://github.com/nswbmw/node-in-debugging/blob/master/5.2%20Elastic%20APM.md)。

## 基本功能
官方文档是相当细致了，使用前推荐阅读。除了基本功能，这里列举一些值得关注的点

- 支持自定义 Node 框架和路由。Agent 记录路由的原理都是 patch 各路由中间件的 match 方法，倘若 SDK 没有对你在用的路由库提供支持，可以选择手动记录或自行 patch。
- 支持主动上报错误 stack，并且帮你在看板上定位异常的来源代码。
- 支持采集 http 请求的 body 参数，默认关闭。一旦开启，可以构成非常强大的日志分析。但不建议在 apm agent 做这种处理，会给监控赋予了太多职能，有需要最好结合全链路 tracing 方案使用额外的 logger agent。
- 过滤敏感信息，根据请求头、或自定义维度。
- 支持定制 Transaction, Span， 额外的 custom 数据。
- 性能优化指南，结合自身业务需要，调整采样率、上报频率、请求体的限制。
- 支持 opentracing
- 支持 kubernetes

## 数据上报

首先我们简称 Elastic APM client 为 agent。agent 到日志采集服务 apm-server 的通讯方式为 http 或 https。请求方法被封装到了 `elastic-http-client` 模块，负责将 Transaction, Error, Metric, Span 这类指标发送到 apm-server，并且还包含格式检查、过长的信息截断的功能。

## 目录结构

相比商业 APM 项目，elastic-apm-node 结构非常简洁。

基本的目录信息如下
- lib
  - filters
  - instrumentation
    - module
  - metrics
    - platform
  - middleware

除了 filters 和 middleware 服务于内部功能，分别用于过滤敏感请求头(auth、cookie) 和异常捕获，剩下的就是核心功能  `Transaction`、`Error`、`Metric` 所在目录了。

接下来，我将围绕这三大功能进行介绍。

# 核心功能

## Error
通常错误日志包含了哪一行代码报错，但只有翻看代码才能确认究竟哪段代码出现问题。APM 提供了可以在看板界面直接确认异常代码的解决方案。

核心代码如下，目的是为了在 Error 对象中通过 __error_callsites 属性获取到 callsites 信息。

```js
var formatter = require('./lib/node-0.10-formatter')

var orig = Error.prepareStackTrace
Error.prepareStackTrace = function (err, callsites) {
  Object.defineProperty(err, '__error_callsites', {
    enumerable: false,
    configurable: true,
    writable: false,
    value: callsites
  })

  return (orig || formatter)(err, callsites)
}

module.exports = function (err) {
  err.stack
  return err.__error_callsites
}
```

上面代码中只有一个不常见的方法 `Error.prepareStackTrace`，并且在 Node.js 的 API 中找不到，因为它实际是 V8 暴露的接口。

```js
Error.prepareStackTrace(error, structuredStackTrace)
```

这个接口常常被用来格式化错误信息，`structuredStackTrace` 包含了一组 CallSite 对象，CallSite 对象支持的方法有：getThis, getTypeName, getFunction, getFunctionName, getMethodName, getFileName, getLineNumber, getColumnNumber, getEvalOrigin, isToplevel, isEval, isNative, isConstructor, isAsync, isPromiseAll, getPromiseIndex

因此借助 CallSite 可以拿到 Error 抛出的文件、行列位置。

- `getFileName`: if this function was defined in a script returns the name of the script
- `getLineNumber`: if this function was defined in a script returns the current line number
- `getColumnNumber`: if this function was defined in a script returns the current column number

最后通过 `source-map` 模块的缓存，获取执行前后的代码。

处理 Error stack 的意义对 JS 直接编写的项目意义可能不是那么大，但假如开发者使用了 TS、或其他原因使生产环境的代码经过了一定编译，这时直接抛出的 Error stack 信息对开发者相当不友好。特定场景下 source-map 的代码映射变得至关重要。

默认地，Elastic APM 只记录 uncaughtException 和一小部分内部 patch 代码的错误。如果有较强的查错需求，得主动在业务中调用 `agent.captureError` 方法记录异常。

另外，若项目有特殊异常上报等原因需要监听 uncaughtException 事件，**应当在 agent start() 之前覆盖 [agent.handleUncaughtExceptions](https://www.elastic.co/guide/en/apm/agent/nodejs/current/agent-api.html#apm-handle-uncaught-exceptions) 方法**，这样才能使其默认的捕获后 exit 的处理失效，以免进程在任务执行结束之前被 APM 的监听器强制结束。

## Metric

一般来说，Node.js 原生暴露的接口足够对进程性能的基本状况有所判断了，但 APM 系统总是希望监控更详细的信息。

尤其是系统 CPU、内存占用率的走势图。一部分探针选择用纯 JS 计算，另一部分探针选择使用 C++ 获取/计算。使用 C++ 的库一般还会获取更复杂的指标，如 [appmetrics](https://github.com/RuntimeTools/appmetrics) 会获取一部分 GC、Event loop 信息（然而 GC 耗费占比的监控在 Node.js Runtime 下无法实现，信息来自：[关于Nodejs的性能监控思考？ - hyj1991的回答 - 知乎](https://www.zhihu.com/question/315261661/answer/637417008)）

Elastic APM 是相对小清新的一派，如果发现当前服务环境 `process.platform` 是 Linux，它会从/proc/ 目录定时获取系统性能快照，无需额外计算，如果是其他系统，再使用 JS 通过算法计算。

> proc 文件的描述可以查看 linux 文件系统文档 https://github.com/torvalds/linux/blob/master/Documentation/filesystems/proc.txt

- /proc/meminfo: 记录系统内存信息，用来获取两个指标：MemAvailable 和 MemTotal。对应 `os.totalmem()` 和 `os.freemem()`。
- /proc/stat: 记录 CPU 活动信息，用来获取两个指标：cpuTotal 和 cpuUsage。这一步用 Node.js 计算略麻烦，需要定时缓存 `os.cpus()` 的 `times.total` `times.idle`指标。
- /proc/self/stat: 不同于前面两个记录系统级信息的文件，此文件记录了当前进程的所有活动信息。用来获取进程 CPU 使用率和 RSS 内存。对应 `processTop.cpu().percent / cpus.length` 和 `process.memoryUsage().rss`。

## Transaction
Elastic APM 中的事务，类似于 opentracing 中的 Span，但把一个请求中所有的 Span 抽象为一个概念。

Transaction 实现的基础是各种代码钩子。

### patch
通过 patch ，做一些信息采集，例如 Koa 框架的。
```js
module.exports = function (koa, agent, { version, enabled }) {
  if (!enabled) return koa

  agent.setFramework({ name: 'koa', version, overwrite: false })

  return koa
}
```

```js
shimmer.wrap(Router.prototype, 'match', function (orig) {
  return function (_, method) {
    var matched = orig.apply(this, arguments)

    if (typeof method !== 'string') {
      agent.logger.debug('unexpected method type in koa-router prototype.match: %s', typeof method)
      return matched
    }

    if (Array.isArray(matched && matched.pathAndMethod)) {
      const layer = matched.pathAndMethod.find(function (layer) {
        return layer && layer.opts && layer.opts.end === true
      })

      var path = layer && layer.path
      if (typeof path === 'string') {
        var name = method + ' ' + path
        agent._instrumentation.setDefaultTransactionName(name)
      } else {
        agent.logger.debug('unexpected path type in koa-router prototype.match: %s', typeof path)
      }
    } else {
      agent.logger.debug('unexpected match result in koa-router prototype.match: %s', typeof matched)
    }

    return matched
  }
})
```

上面的 patch 配合 `require-in-the-middle` 模块，完成了对各框架的包装。

### async-hook
利用 async-hook 实现记录整串请求，来看两个代码片段。

首先是基于 async-hook 封装了 Instrumentation 的 `currentTransaction` 方法，使异步操作中随时可以拿到当前 async scope id 下的 Transaction 实例。

```js
const asyncHooks = require('async_hooks')
module.exports = function (ins) {
  const asyncHook = asyncHooks.createHook({ init, before, destroy })
  const contexts = new WeakMap()

  const activeTransactions = new Map()
  Object.defineProperty(ins, 'currentTransaction', {
    get () {
      const asyncId = asyncHooks.executionAsyncId()
      return activeTransactions.get(asyncId) || null
    },
    set (trans) {
      const asyncId = asyncHooks.executionAsyncId()
      if (trans) {
        activeTransactions.set(asyncId, trans)
      } else {
        activeTransactions.delete(asyncId)
      }
    }
  })
  // ...
}
```

下面是 currentTransaction 的一处应用

```js
Instrumentation.prototype.bindFunction = function (original) {
  if (typeof original !== 'function' || original.name === 'elasticAPMCallbackWrapper') return original

  var ins = this
  var trans = this.currentTransaction
  var span = this.currentSpan
  if (trans && !trans.sampled) {
    return original
  }

  return elasticAPMCallbackWrapper

  function elasticAPMCallbackWrapper () {
    var prevTrans = ins.currentTransaction
    ins.currentTransaction = trans
    ins.bindingSpan = null
    ins.activeSpan = span
    if (trans) trans.sync = false
    if (span) span.sync = false
    var result = original.apply(this, arguments)
    ins.currentTransaction = prevTrans
    return result
  }
}
```

async hook 是 Node.js 8 以后出现的概念，为了兼容旧版本，Elastic APM 借助 `async-listener` 模块做了一些兼容，尽管 Elastic APM 官方不推荐使用低版本 Node.js 接入。

虽然 async hook 更进一步可以帮助优化异步调用栈，改善异步 Error 信息的可读性，但 APM 很难从底层判断哪些异步 CallSite 是用户想保留的，所以没有做这种处理。

### Span Trace
Span 用来记录 db 操作、http、websocket 远程调用等细致操作，Elastic APM 同时记录了调用栈。

我们知道，console.trace() 方法可以用来定位 trace 信息，它实际使用了 V8 Error 暴露的另一个方法 `Error.captureStackTrace(error, constructorOpt)`。

`error` 是用来记录 trace 信息的必传对象，captureStackTrace 会将字符串附加到 error 对象的 stack 属性上。

`constructorOpt` 是用来隐藏底层调用栈的可选函数，用法如下
```js
function MyError() {
  Error.captureStackTrace(this, MyError);
  // Any other initialization goes here.
}
```

#### 小插曲
上面提到的 V8 Error trace API，结合 TJ 的 `callsite` 更容易理解，功能是获取当前的 CallSite 集合。
```js
module.exports = function(){
  var orig = Error.prepareStackTrace;
  Error.prepareStackTrace = function(_, stack){ return stack; };
  var err = new Error;
  Error.captureStackTrace(err, arguments.callee);
  var stack = err.stack;
  Error.prepareStackTrace = orig;
  return stack;
};
```

# 总结

不得不说，和活跃的商业巨头产品相比，Elastic APM 目前的功能支持度存在不少差距。如果想在 APM 专业领域探索，绕不开对 New Relic 源码的学习。 XD

但 Elastic APM Node.js 依然是目前最值得关注的开源探针式监控方案，其 agent 基础功能支持度较好，代码结构也很简单，希望能被更多人使用，帮助它更快成长~

# Reference

- https://github.com/elastic/apm-agent-nodejs
- https://v8.dev/docs/stack-trace-api