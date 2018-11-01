# 模块

* [`[Basic]` 模块机制](#模块机制)
* [`[Basic]` 热更新](#热更新)
* [`[Basic]` 上下文](#上下文)
* [`[Basic]` 包管理](#包管理)

## 常见问题


> <a name="q-hot"></a> 如何在不重启 node 进程的情况下热更新一个 js/json 文件? 这个问题本身是否有问题?

可以清除掉 `require.cache` 的缓存重新 `require(xxx)`, 视具体情况还可以用 VM 模块重新执行.

当然这个问题可能是典型的 [`X-Y Problem`](http://coolshell.cn/articles/10804.html), 使用 js 实现热更新很容易碰到 v8 优化之后各地拿到缓存的引用导致热更新 js 没意义. 当然热更新 json 还是可以简单一点比如用读取文件的方式来热更新, 但是这样也不如从 redis 之类的数据库中读取比较合理.

## 简述

其他还有很多内容也是属于很 '基础' 的 Node.js 问题 (例如异步/线程等等), 但是由于归类的问题并没有放在这个分类中. 所以这里只简单讲几个之后没归类的基础问题.


## 模块机制

node 的基础中毫无疑问的应该是有关于模块机制的方面的, 也即 `require` 这个内置功能的一些原理的问题.

关于模块互相引用之类的, 不了解的推荐先好好读读[官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/modules.html).

其实官方文档已经说得很清楚了, 每个 node 进程只有一个 VM 的上下文, 不会跟浏览器相差多少, 模块机制在文档中也描述的非常清楚了:

```javascript
function require(...) {
  var module = { exports: {} };
  ((module, exports) => {
    // Your module code here. In this example, define a function.
    function some_func() {};
    exports = some_func;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    module.exports = some_func;
    // At this point, the module will now export some_func, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
```

> <a name="q-global"></a> 如果 a.js require 了 b.js, 那么在 b 中定义全局变量 `t = 111` 能否在 a 中直接打印出来?

① 每个 `.js` 能独立一个环境只是因为 node 帮你在外层包了一圈自执行, 所以你使用 `t = 111` 定义全局变量在其他地方当然能拿到. 情况如下:

```javascript

// b.js
(function (exports, require, module, __filename, __dirname) {
  t = 111;
})();

// a.js
(function (exports, require, module, __filename, __dirname) {
  // ...
  console.log(t); // 111
})();
```

> <a name="q-loop"></a> a.js 和 b.js 两个文件互相 require 是否会死循环? 双方是否能导出变量? 如何从设计上避免这种问题?

② 不会, 先执行的导出空对象, 通过导出工厂函数让对方从函数去拿比较好避免. 模块在导出的只是 `var module = { exports: {} };` 中的 exports, 以从 a.js 启动为例, a.js 还没执行完 exports 就是 `{}` 在 b.js 的开头拿到的就是 `{}` 而已.
> bingou

require()加载机制：[`js模块的循环加载`](http://www.ruanyifeng.com/blog/2015/11/circular-dependency.html) [`require()源码解读`](http://www.ruanyifeng.com/blog/2015/05/require.html)

require加载：
- commonjs一个模块就是一个脚本，`require`第一次加载脚本就会整个脚本（加载时执行），然后在内存生成一个对象。以后再执行这个模块就会从这个对象的exports属性上取值，即再次执行require命令也不会再次执行该模块，而是到缓存中取值。
  ```js
  {
    id: '...', //属性名
    exports: { ... }, //模块输出的各个接口
    loaded: true, //表示模块脚本是否已经执行完毕
    ...
  }
  ```
- 一旦出现某个模块被循环加载，只输出已经执行的部分，还未执行部分不会被输出。

es6模块加载：
- es6模块的运行机制是遇到模块加载命令import时，不会去执行模块而是只生成一个引用，到真正需要用到时再到模块中去取值。
- 因为是动态引用，不存在缓存值的问题，模块的值绑定在所在模块，es6不关心是否发生了循环加载，只需根据指向被加载模块的引用去取值；这就需要开发者自己暴增真正取值的时候能够取到，只要引用存在代码就能执行。
  ```js
  //commonjs
  let {stat, exists, readFile} = reuqire('fs');
  
  //等同于
  let _fs = require('fs');
  let stat = _fs.stat;
  let exists = _fs.exists;
  let readfile = _fs.readfile;
  ```
- 上述代码实质是加载了整个fs模块（fs的所有方法），生成一个_fs对象，然后从这个对象上读取3个方法，成为运行时加载，因为只有运行时才能得到这个对象，导致没有办法再编译时做静态优化。
  
  ```js
  import { stat, exists, readFile } from 'fs';
  ```
- es6模块不是对象，而是通过export命令显示指定输出代码，再通过import命令输入。上面的代码实质时从fs模块加载3个方法，**其他方法不加载**。
- 这种加载称为编译时加载或者静态加载，，即es6编译时就可以完成模块的加载，效率要比commonjs模块加载方式高。当然这也导致没法引用es6模块本身，因为它不是对象。
- import命令会被js引擎静态分析，先于模块内其他语句的执行，由于es6模块时编译时加载使得静态分析成为可能，因此可以使用treeShaking进行优化代码。
  
require的路径处理
-  脚本文件 /home/ry/projects/foo.js 执行了 require('bar'),依次搜索一下路径确定bar目录
    ```js
    /home/ry/projects/node_modules/bar
    /home/ry/node_modules/bar
    /home/node_modules/bar
    /node_modules/bar
    ```
- node先将bar当成文件名依次搜索bar/bar.js/bar.json/bar.node,只要一个成功就返回
- 如果都不成功则说明bar可能是目录名，尝试加载
    ```js
    bar/package.json（main字段）
    bar/index.js
    bar/index.json
    bar/index.node
    ```
> bingou

另外还有非常基础和常见的问题, 比如 module.exports 和 exports 的区别这里也能一并解决了 exports 只是 module.exports 的一个引用. 没看懂可以在细看我以前发的[帖子](https://cnodejs.org/topic/5734017ac3e4ef7657ab1215).

再晋级一点, 众所周知, node 的模块机制是基于 [`CommonJS`](http://javascript.ruanyifeng.com/nodejs/module.html) 规范的. 对于从前端转 node 的同学, 如果面试官想问的难一点会考验关于 [`CommonJS`](http://javascript.ruanyifeng.com/nodejs/module.html) 的一些问题. 比如比较 `AMD`, `CMD`, [`CommonJS`](http://javascript.ruanyifeng.com/nodejs/module.html) 三者的区别, 包括询问关于 node 中 `require` 的实现原理等.

## 热更新

从面试官的角度看, `热更新` 是很多程序常见的问题. 对客户端而言, 热更新意味着不用换包, 当然也包含着 md5 校验/差异更新等复杂问题; 对服务端而言, 热更新意味着服务不用重启, 这样可用性较高<del>同时也优雅和有逼格</del>. 问的过程中可以一定程度的暴露应聘程序员的水平.

从 PHP 转 node 的同学可能会有些想法, 比如 PHP 的代码直接刷上去就好了, 并没有所谓的重启. 而 node 重启看起来动作还挺大. 当然这里面的区别, 主要是与同时有 PHP 与 node 开发经验的同学可以讨论, 也是很好的切入点.

在 Node.js 中做热更新代码, 牵扯到的知识点可能主要是 `require` 会有一个 `cache`, 有这个 `cache` 在, 即使你更新了 `.js` 文件, 在代码中再次 `require` 还是会拿到之前的编译好缓存在 v8 内存 (code space) 中的的旧代码. 但是如果只是单纯的清除掉 `require` 中的 `cache`, 再次 `require` 确实能拿到新的代码, 但是这时候很容易碰到各地维持旧的引用依旧跑的旧的代码的问题. 如果还要继续推行这种热更新代码的话, 可能要推翻当前的架构, 从头开始从新设计一下目前的框架.

不过热更新 json 之类的配置文件的话, 还是可以简单的实现的, 更新 `require` 的 `cache` 可以实现, 不会有持有旧引用的问题, 可以参见我 2 年前写着玩的[例子](https://www.npmjs.com/package/auto-reload), 但是如果旧的引用一直被持有很容易出现内存泄漏, 而要热更新配置的话, 为什么不存数据库? 或者用 `zookeeper` 之类的服务? 通过更新文件还要再发布一次, 但是存数据库直接写个接口配个界面多爽你说是不是?

所以这个问题其实本身其实是值得商榷的, 可能是典型的 [`X-Y Problem`](http://coolshell.cn/articles/10804.html), 不过聊起来确实是可以暴露水平.

## 上下文

如果你已经了解 ①② 那么你也应该了解, 对于 Node.js 而言, 正常情况下只有一个上下文, 甚至于内置的很多方面例如 `require` 的实现只是在启动的时候运行了[内置的函数](https://github.com/nodejs/node/tree/master/lib). 

每个单独的 `.js` 文件并不意味着单独的上下文, 在某个 `.js` 文件中污染了全局的作用域一样能影响到其他的地方.

而目前的 Node.js 将 VM 的接口暴露了出来, 可以让你自己创建一个新的 js 上下文, 这一点上跟前端 js 还是区别挺大的. 在执行外部代码的时候, 通过创建新的上下文沙盒 (sandbox) 可以避免上下文被污染:

```javascript
'use strict';
const vm = require('vm');

let code =
`(function(require) {

  const http = require('http');

  http.createServer( (request, response) => {
    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('Hello World\\n');
  }).listen(8124);

  console.log('Server running at http://127.0.0.1:8124/');
})`;

vm.runInThisContext(code)(require);
```

这种执行方式与 eval 和 Function 有明显的区别. 关于 VM 更多的一些接口可以先阅读[官方文档 VM (虚拟机)](https://nodejs.org/dist/latest-v6.x/docs/api/vm.html)

讲完这个知识点, 这里留下一个简单的问题, 既然可以通过新的上下文来避免污染, 那么`为什么 Node.js 不给每一个 `.js` 文件以独立的上下文来避免作用域被污染?` <del>(反应不过来的同学还是别投简历了, 微笑脸)</del>


## 包管理


整理中...

为什么我装了全局, 但是提示我 not found

npm
yarn

锁版本

lerna：一个用户管理多个包模块的工具。

left-pad事件

greenkeeper 等
