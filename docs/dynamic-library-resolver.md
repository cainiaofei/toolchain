# 要解决的问题

* 给运行时库一个名字，通过名字找到运行时环境里的库
* 在运行时库的多个库里定位一个合适的

# 解决方案

动态链接库不会直接从远程加载，一般都要保留在本地的硬盘上。每个机器的安装动态链接库的位置不同，所以一般是不会用硬盘的文件绝对路径去加载动态链接库的。标识并定位的解决方案就是提供一个安装环境无关的动态链接的过程。

## 构成

* dynmaic library name：动态链接库的名字
* ld library path：加载链接库要在哪里查找。最常用的是动态链接发起的文件所在路径，做为查找相对路径的参照
* resolver：根据 dynamic library name 从 ld library path 里查找动态链接库

# 解决方案案例

JavaScript 的几种动态链接库 linker 的解析名字的行为，如下表所示

| linker | 绝对路径（/library.js） | 相对路径（./library.js） | bare specifier（library.js 或者 library） |
| --- | --- | --- | --- |
| global | http 或者 https 根据绝对路径 | 相对加载 html 页面的 url | 当相对路径处理 |
| AMD | http 或者 https 根据绝对路径 | 相对加载 html 页面的 url | config 里的 baseUrl |
| CJS | 从文件系统的绝对路径加载 | 相对当前js文件路径加载 | node_modules 链，NODE_PATH 兜底 |
| ES6 |  http 或者 https 根据绝对路径 | 相对加载 html 页面的 url | 不支持（import maps 的标准尚处于草案阶段） |
| System.register | | | |

## 传统浏览器

* dynamic library name：http/https 绝对 url 或者相对 url
* ld library path：加载 script 所在网页的 url
* resolver：浏览器自身

相对路径

```html
// http://localhost/a/b.html
<html>
<head>
<script src="c/d.js"></script>
</head>
...
</html>
```

`c/d.js` 相对 `http://localhost/a/b.html` 解析出来就是 `http://localhost/a/c/d.js`

提供 `//url` 做为相对参照。如果当前网页是 https，则 `//` 相当于 `https://`

```html
// https://localhost/a/b.html
<html>
<head>
<script src="//some-cdn.com/c/d.js"></script>
</head>
...
</html>
```

解析出来就是 `https://some-cdn.com/c/d.js`

## AMD 的代表 require.js

* dynamic library name：`requirejs(['app/main'])` 里指定的名字
* ld library path：config 里的 baseUrl
* resolver：require.js

```
requirejs.config({
    baseUrl: 'lib',
    paths: {
        app: '../app'
    }
});
```

对于 baseUrl 而言，如果使用了以下几种写法

* 绝对路径：/library.js
* 相对路径：./library.js
* 带后缀：library.js

只有在 rqeuire('library') 的时候，才会使用 baseUrl 进行相对路径的查找。

## CJS 的代表 nodejs

* dynamic library name：nodejs 的 resolver 支持三种指定 library_name 的方式
  * 绝对路径：/opt/library.js
  * 相对路径：./library.js
  * bare specifier：library.js
* ld library path：NODE_PATH 环境变量
* resolver：nodejs自身

| 类型 | 例子 | resolve 过程 |
| --- | --- | --- |
| 绝对路径 | /opt/library.js | 相对硬盘根目录查找文件 |
| 相对路径 | ./library.js | 相对当前文件所在目录 |
| bare module specifier | library.js | 先查找 node_modules，再查找 NODE_PATH 环境变量指定的路径 |

举例说明 bare module spcifier，有如下的目录结构

* /opt
  * level1
    * node_modules
      * library
        * package.json
        * library.js
    * level2
      * executable.js

/opt/level1/level2/executable.js 引用 `require('library')` 会使用 /opt/level1/node_modules/library/package.json 定义的动态链接库

会尝试的 node_modules 路径包括

* ./node_moduels 当前目录下的 node_mdoules 目录
* ../node_modules 父目录的 node_modules 目录
* 省略
* /node_modules 根目录的 node_modules 目录

如果在当前目录一直到根目录的 node_modules 里都找不到指定的 dynamic library name，则会使用 NODE_PATH 去查找。例如 NODE_PATH 如下 `/usr/lib/node_modules1:/usr/lib/node_modules2`


* /usr
  * lib
    * node_modules1
    * node_modules2
      * library
        * package.json
        * library.js
* /opt
  * level1
    * level2
      * executable.js

```
export NODE_PATH="/usr/lib/node_modules1:/usr/lib/node_modules2" 
node /opt/level1/level2/executable.js
```

在 /usr/lib/node_modules1 里没找到，就会去 /usr/lib/node_module2 里找，然后找到了

## 支持 ES6 Module 的浏览器

* dynamic library name：http/https 绝对 url 或者相对 url
* ld library path：加载 script 所在网页的 url
* resolver：浏览器自身

```html
// http://localhost/index.html
<html> 
<head>
<script type="module">
import * as lib from 'http://localhost/library.js'
</script>
</head>
<body> 
</body>
</html>
```

```js
// http://localhost/library.js
console.log('i am the library')
```

用浏览器访问 `http://localhost/index.html` 输出到浏览器控制台

```
i am the library
```

基本上来说，ES6 Module对动态链接库的命名，以及resolve过程，和传统浏览器加载 script 标签是一样的。

## 支持 ES6 Module 的 nodejs

用 `import './library.js'` 代替 `require('./library.js')`，resolve的行为基本一致。

但是如果使用 NODE_PATH 去做 `import 'library'` 是不支持的，会出现提示

```
(node:1031) ExperimentalWarning: The ESM module loader is experimental. 
Error [ERR_MODULE_RESOLUTION_LEGACY]: library not found by import in file:///home/taowen/test/level1/level2/executable.mjs. Legacy behavior in require() would have found it at /home/taowen/test/opt/node_modules/library/library.js 
    at search (internal/modules/esm/default_resolve.js:39:15)
    at Loader.resolve [as _resolve] (internal/modules/esm/default_resolve.js:65:11)
    at Loader.resolve (internal/modules/esm/loader.js:58:33)
    at Loader.getModuleJob (internal/modules/esm/loader.js:113:40)
    at ModuleWrap.promises.module.link (internal/modules/esm/module_job.js:32:40)
    at link (internal/modules/esm/module_job.js:31:36)
```

因为 ES6 Module 规范还没有定义如何处理所谓的 `bare specifier`，所以 node.js 为了和标准保持一致，把自己实现的 NODE_PATH 给阉割掉了。




