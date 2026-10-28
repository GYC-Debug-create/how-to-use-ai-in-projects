# TEACHING_RULES.md

## 1. 讲解总原则

用户不是完全不会编程，但前端和 JS 很弱。

讲解顺序必须是：

1. 当前代码逐行解释；
2. 这行代码连接到 template / script 的哪里；
3. 用户操作后它什么时候执行；
4. 浏览器里会发生什么变化；
5. （已脱敏）

不要反过来先讲概念。

## 2. 必须贴代码讲的点

遇到以下内容必须结合当前代码解释：

* ref
* reactive
* v-model
* v-if
* v-show
* @click
* @submit.prevent
* async / await
* try / catch
* 函数引用
* Axios response.data
* error.response?.data?.message
* router.push
* beforeEach

## 3. 禁止的讲解方式

禁止：

* “Vue 响应式本质上是……”，但不说当前哪行代码触发页面变化。
* “async/await 是 Promise 语法糖”，但不说当前 response 是怎么来的。
* “v-model 是双向绑定”，但不说 form.username 什么时候变。
* “catch 统一处理异常”，但不说 error.response 什么时候存在。
* （已脱敏）

## 4. 推荐解释格式

示例：

代码：

```js
isLogin.value = !isLogin.value
```

解释：

* `isLogin` 是 ref 包装的布尔值；
* `.value` 是它真正保存的值；
* `!isLogin.value` 会把 true 变 false，把 false 变 true；
* template 中的 `{{ isLogin ? '登录' : '注册' }}` 依赖这个值；
* 所以用户点击“去注册/去登录”后，页面文字会跟着变。

## 5. 用户追问处理

如果用户表示“没懂”，必须缩小解释范围，只解释当前一行或当前一个变量。

不要扩大到新概念。
