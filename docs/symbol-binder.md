# 要解决的问题

如何引用动态链接库指定的符号

# 解决方案

动态链接库提供了很多个函数，类或者常量之类的东西，我们统称为symbol。引用了具体的符号才能开始使用。这个需要定义的地方对符号进行导出，使用的地方对符号进行导入。由 binder 进行符号的撮合绑定。

## 构成

* exported symbol: 动态链接库导出的符号
* imported symbol：导入的符号
* binder：把导入的符号和导出的符号绑定到一起

# 解决方案案例

## 传统浏览器

传统浏览器没有 binder。全局共享一个namespace（window对象），动态链接库导出和导入通过读写全局变量实现。

## 支持 ES6 Module 的浏览器或者 nodejs

* exported symbol
  * `export function my_func() {}` 定义和export合一
  * `function my_func() {}; export {my_func as my_exported_func}` 定义和export分离
  * `export default function my_func() {}` export成为特殊的符号`default`
* imported symbol
  * `import './library.mjs'` 只加载动态链接库，但是并不导入符号
  * `import {my_func} from './library.mjs'` 导入动态链接库中的指定符号 my_func
  * `import my_func from './library.mjs'` 把导出的 `default` 符号，导入并重命名为 my_func
  * `import * as lib from './library.mjs'` 把动态链接库整体导入为 lib
* binder：浏览器或者nodejs

ES6 Module 的导入的是引用，如果源对象发生了变化，导入的符号也会跟着变化

```js
// /opt/library.mjs
export let my_var = 3 
export function increase_my_var() {
  my_var++
}
```

```js
// /opt/executable.mjs
import {my_var, increase_my_var} from './library.mjs' 

console.log(my_var)
increase_my_var()
console.log(my_var)
```

```
node --experimental-modules executable.mjs
// Output:
// 3
// 4
```

## nodejs

* exported symbol
  * `exports.my_func = function() {}` 定义和export合一
  * `function my_func() {}; exports.my_exported_func = my_func` 定义和export分离
  * `module.exports = function() {}` 只导出一个符号，导入的时候也无需选择
* imported symbol
  * `require('./library.js')` 只加载动态链接库，但是并不导入符号
  * `const my_func = require('./library.js').my_func` 导入链接库中的指定符号 my_func
  * `const lib = require('./library.js')` 把动态链接库整体导入为 lib
binder：nodejs

nodejs 的 require 语法的导入是拷贝值的行为，和ES6 Module不同

```js
// /opt/library.js
exports.my_var = 3 
exports.increase_my_var = function() {
  exports.my_var++
}
```

```js
// /opt/executable.js
const {my_var, increase_my_var} = require('./library.js') 
console.log(my_var)
increase_my_var()
console.log(my_var)
console.log(require('./library.js').my_var)
```

```
node executable.js
// Output:
// 3
// 3
// 4
```

我们可以看到导入这份 my_var 没有改变，而原始的那份 my_var 已经改变了。

定义唯一导出符号需要重新定义 `module.exports`

```js
// /opt/library.js
module.exports = function() {
  console.log('library called')
}
```

```js
// /opt/executable.js
const my_func = require('./library.js')
my_func()
```

```
node executable.js
// Ouptut:
// library called
```


