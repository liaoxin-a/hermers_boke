---
title: 框架是 Web 的"打补丁"层:为什么 HTML/CSS/JS 永远慢半拍
date: 2026-09-02
tags: [Web平台, 前端架构, CSS, JavaScript, 框架]
author: Hermes
---

## 引子：一段你写过的"补丁"

2021 年，你接到一个很普通的需求：一张卡片，当它所在的容器变宽时，从单列自动切成双列。今天写起来只是几行 CSS；但当时 `@container` 还不存在——容器查询要等到 2022 年 8 月的 Chrome 105 才落地，而社区从 2010 年代就开始呼吁这种能力，前后近十年。于是你只能打开编辑器，写一段 JavaScript 来"打补丁"：

```html
<!-- 补丁形态:2021 年,容器查询落地之前 -->
<div class="card">
  <div class="card-grid">…</div>
</div>

<script>
  const card = document.querySelector('.card');
  const grid = document.querySelector('.card-grid');

  // 没有 @container,就用 ResizeObserver 模拟"容器断点"
  function sync() {
    grid.classList.toggle('wide', card.clientWidth > 700);
  }

  new ResizeObserver(sync).observe(card);
  sync(); // 处理初始状态
</script>

<style>
  .card-grid.wide { display: grid; grid-template-columns: 1fr 1fr; }
</style>
```

这段代码平淡无奇，但它是一整类现象的缩影：HTML/CSS/JS 这个平台，从诞生起就被要求"永不推倒重来"，只能以十年为单位缓慢进化；框架则以两年为单位快速迭代，专门填补平台"来不及给"的能力。平台与框架之间的这条时差，就是前端世界里那场永不停歇的"打补丁"循环。

## 一、平台为什么不能推倒重来

别的技术可以 v2 重写：框架靠 semver 发大版本，breaking change 不过是发布说明里的一行。浏览器不行。Web 平台的"宪法"写在 HTML5 设计原则里：**Priority of Constituencies（冲突时用户 > 作者 > 实现者 > 规范制定者 > 理论纯洁性），以及 "Evolution Not Revolution——进化而非革命，不是推倒重来"**。改动一条行为，可能弄坏数十亿存量页面，所以标准委员会极端保守。2013 年，Extensible Web Manifesto 的发布说明帖（共同发起人 François REMY，2013-06-11）把原因说得直白：

> "Once something shipped in a browser, removing it is painful (at least) and often impossible… But, sometimes, web developers can't wait."
> ——一旦发布就很难撤回；可是，开发者等不起。

那句"等不起"，正是框架存在的全部理由。

**为什么重要**：理解了"平台必须慢"，你才不会反复用"为什么原生这么难用"折磨自己。CSS 的全局级联、HTML 的容错解析、JS 的单线程模型——这些"意外复杂度"不是可以修掉的 bug，而是"1991 年的第一张网页今天仍能渲染"的代价，是 30 年兼容债的利息。框架不是来消灭复杂度的，它只是来垫付利息的。

## 二、打补丁循环：十年一吸收，两年一补丁

把时间线拉长，规律清晰得近乎残酷：**补丁总是先到，平台总是迟到。**

| 平台缺口 | 社区/框架的"补丁" | 平台最终吸收 |
|---|---|---|
| IE6 时代 DOM/事件/选择器各写各的 | jQuery（2006，"write less, do more"）抹平差异 | querySelectorAll、classList、fetch（约 2009–2015 补齐） |
| JS 没有模块系统 | CommonJS、webpack 打包器 | ES Modules，标准化近 10 年，2018 年全主流浏览器支持 |
| CSS 没有变量与高级布局 | Sass（2006）、BEM 命名方法论 | 自定义属性、Flexbox（2009→2018 CR，约 9 年）、Grid |
| 容器要响应自身宽度 | ResizeObserver + 手写断点 | Container Queries（Chrome 105，2022-08） |
| 回调嵌套地狱 | Promise 库（Bluebird 等） | 原生 Promise（ES2015）、async/await（ES2017） |

几个数字值得记住：jQuery 在 2006 年抹平了 IE6 时代的差异，平台花了近十年才补齐对应的原生 API；ES Modules 从 CommonJS 与打包器时代走到全浏览器支持，标准化花了近十年（Lin Clark 称之为 "nearly 10 years of standardization work"）；Flexbox 从 2009 年 7 月启动到 2018 年 11 月成为 Candidate Recommendation，约九年。

慢，还因为平台不是一家公司说了算：TC39 与 CSSWG 需要多家浏览器厂商达成共识，一家反对就可能搁浅整个方案——Safari 至今拒绝 customized built-ins（2016 年争论至今），Houdini 想把"CSS 引擎暴露给开发者"，但 Layout 与 Parser API 至今没有任何浏览器实现。浏览器厂商直到 2022 年的 Interop 2022 才第一次联手，用共享测试套件系统性修补互操作差距——此前"行为不一致"的土壤，是真实且长期的。

**为什么重要**：这张表是判断工具命运的罗盘。凡是针对"平台缺口"的补丁——模块化、响应式、异步——大概率会被平台吸收，这类债可以还清；凡是针对"上一个框架的债"的补丁，则是另一回事（见第四节）。

## 三、两个标本：意外复杂度与正在被吸收的补丁

**标本一：CSS 级联的"特权战争"。** 级联是 CSS 最强大的机制，也是最反直觉的机制——一个元素最终长什么样，由选择器的特异性（specificity）、书写顺序、`!important` 共同决定，而这些全是隐式全局规则：

```html
<style>
  button.btn     { background: #2563eb; } /* 特异性 (0,1,1) */
  .btn.primary   { background: #16a34a; } /* (0,2,0),赢过上一行 */
  #submit        { background: #dc2626; } /* (1,0,0),又赢过前两行 */
  button         { background: #6b7280 !important; } /* 通杀 */
</style>
<button id="submit" class="btn primary">提交</button>
<!-- 最终颜色是灰。答案藏在四条规则的特权大小里 -->
```

同一段 HTML，四条规则都声称拥有背景色：class 数量压过单个 class，ID 又压过 class，`!important` 最后通杀。调样式失效时，你查的不是"哪行写错了"，而是"谁的特权更大"。

**为什么重要**：这正是 BEM、CSS Modules、Tailwind 这类"补丁"的真实动机——它们不是审美偏好，而是把特异性压平到可预测的区间，给一个全局隐式系统"上保险"。原生 CSS 至今没有类名命名空间；直到 2023–2025 年，`@scope` 才在三大引擎全面落地（Chrome 118、Safari 17.4、Firefox 146），提供原生样式作用域——和容器查询一样，这类"作用域补丁"正迎来被平台吸收的终点。

**标本二：容器查询——补丁被平台吸收的全过程。** 回到开头的需求，2021 年的写法是命令式的；等平台落地后，同样的效果变成一句声明：

```css
.card { container-type: inline-size; } /* 声明:我是容器 */

@container (width > 700px) {
  .card-grid { display: grid; grid-template-columns: 1fr 1fr; }
}
```

命令式补丁要自己管理监听、初始状态与清理；平台原语则把脏活全交给浏览器。**为什么重要**：原生到来之后，那段 JS 可以直接删掉——这就是"平台缺口补丁"与"永久债"的区别：前者有还清的一天。

## 四、平台吸收补丁后，框架为什么不消失

最反直觉的事实在这里：平台真的吸收了补丁，框架也没有消失——**它会再叠一层，去补"上一轮补丁制造的债"**。SSR 解决了首屏与 SEO，却带来整页注水（hydration）的成本，于是 2020 年有了 Islands 架构（部分水合，Astro 等）；虚拟 DOM 解决了命令式 DOM 的混乱，却引入 diff 开销，于是 Svelte 转向编译时、Qwik 提出 resumability、React 转向编译器；客户端渲染把整站逻辑塞进浏览器，于是又出现 React Server Components，以及 htmx 这类"回到超媒体"的反思。补丁不是消失了，而是向上再叠一层。

何况还有生态的自我强化：招聘 JD 写着 React，组件库、教程、团队经验全围着框架转，替换成本高到不现实。"框架不会消失"这件事，根本不需要技术解释。

**为什么重要**：它解释了一个令人厌倦的现象——为什么每隔几年就要学一套新范式，而上一套还没捂热。不是开发者健忘，而是平台的结构性时差，逼出了这种节奏。

## 结语：把补丁当终点，才是问题

补丁不是错误。在一个拒绝停机的平台上，补丁是唯一能用的维修方式；真正的问题，是把补丁当成了终点。落到日常，三条建议：

1. 每次引入新框架或库，先问一句：它在补"平台缺口"，还是在补"上一个框架的债"？前者大概率会被平台吸收（容器查询、`:has()` 落地后，成堆 polyfill 与绕行代码都能删），债可还；后者往往意味着整层技术栈的更迭，要谨慎。
2. 优先投资平台原语：ES Modules、CSS 布局、fetch、Web Components 的边界。它们的半衰期以十年计，框架语法以两年计。
3. 盯住标准轨道：缺口一旦进入候选推荐标准（Candidate Recommendation）或主流浏览器开始实现，用 polyfill 过渡即可，不必再上一个重框架。

平台很慢，但它从不缺席——前提是你算得清：手里的补丁，是过渡，还是终局。
