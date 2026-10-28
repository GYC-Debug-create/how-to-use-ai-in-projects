---
name: douyin-lite-frontend-ai-coding
description: 用于 douyin_lite 前端项目的 AI 协作规则。用户有 Java/C++/后端基础，但 JS/Vue 基础弱。优先保证可运行小步、代码贴身讲解和浏览器验证。
---

# Douyin Lite Frontend AI Coding Skill

## 1. 用户背景

用户有 Java / C++ / 后端基础，但 JavaScript、Vue、前端运行机制较弱。

不要默认用户已经理解：

* ref / reactive
* v-model / v-if / v-show
* @click / @submit.prevent
* async / await / try / catch
* 函数引用
* Axios 响应结构
* Router 路由守卫
* 浏览器 DOM 更新机制

## 2. 必读文件顺序

每轮开始前按顺序读取：

1. SKILL.md
2. FRONTEND_PROGRESS.md
3. CURRENT_TASK.md
4. TEACHING_RULES.md

不要默认读取 FUTURE_OPTIMIZATIONS.md。该文件只在阶段收尾、重构或用户明确要求优化时读取。

## 3. 硬性规则

1. 每一步改代码后，页面必须尽量保持可运行、可验证。
2. 禁止提交会导致白屏的中间态，除非修改前明确说明“这一步会暂时不可运行”并解释原因。
3. “小步”不是写半截代码，而是一个能看到效果、能验证的最小闭环。
4. 不要一轮同时改 UI、API、Router、状态管理多个层次。
5. 不要从 LoginView.vue 直接跳到 router/index.js，除非当前文件已经讲解并验证完成。
6. 不要使用容易混淆的变量名。例如不要把函数引用命名为 api，优先用 authRequest、submitAuthRequest、currentAuthAction。
7. 临时演示代码必须在代码注释和说明里明确标注“临时演示，后续删除”。
8. （已脱敏）
9. 不要先讲大段抽象概念，例如响应式系统、Observer 模式、async/await 原理，除非用户明确要求。
10. 用户问“这行代码干了什么”时，必须只解释这行在当前代码链路里的作用。
11. 不要频繁催用户“继续”。每一步结束后先确认当前代码是否理解。
12. 不执行任何 git add / commit / push。

## 4. 每次修改前必须输出

* 本次准备修改哪个文件；
* 为什么必须改这个文件；
* 本次只完成什么最小功能；
* 修改后浏览器应该看到什么；
* 是否可能导致页面暂时不可运行。

## 5. 每次修改后必须输出

* 本次改了哪些文件；
* 新增/修改了哪些变量、函数、模板语句；
* template 中每个动态语句对应 script 中哪个变量或函数；
* 用户操作后代码从哪一行流向哪一行；
* 浏览器应该如何验证；
* 当前还没实现什么；
* 下一步准备做什么。

## 6. 登录注册页面推荐推进顺序

LoginView.vue 必须按以下顺序推进：

1. 静态 HTML 骨架：不引入 Vue 动态变量，页面必须能显示。
2. 加入 ref / reactive 状态：解释 isLogin、form、errorMessage。
3. 加入 v-model：解释输入框如何同步 form.username / form.password。
4. 加入登录/注册切换：解释 isLogin 和 toggleMode。
5. 加入 errorMessage + v-if + 非空校验：必须能看到红字出现/消失。
6. 接入 login/register API：解释 async/await、try/catch、token 保存和跳转。
7. 浏览器验证完整登录链路。
8. 当前页面复盘完成后，再进入 router/index.js 路由守卫。
