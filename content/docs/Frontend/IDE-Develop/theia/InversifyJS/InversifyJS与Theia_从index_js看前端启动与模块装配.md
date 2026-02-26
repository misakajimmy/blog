---
title: "InversifyJS 与 Theia：从 index.js 看前端启动与模块装配"
weight: 13
---

> 前面几篇都是站在“纯 InversifyJS”的视角看依赖注入，这篇换个角度，  
> **直接拿 Theia 的 `index.js` 启动脚本拆解一下，看看 InversifyJS 在真实 Theia 应用里到底干了些什么。**

## 这个 index.js 在 Theia 里扮演什么角色？

先整体看一眼，下面是一个直接内嵌在本页的 `index.js` 预览：

<iframe src="/code/theia/index.html"
  style="width:100%; height:500px; border:1px solid #ddd; border-radius:4px; overflow:hidden;"
  title="Theia index.js source preview"
  loading="lazy"
></iframe>

大致结构可以简化理解成这样：

```js
require('reflect-metadata');
const { Container } = require('@theia/core/shared/inversify');
// ...
module.exports = (async () => {
  const container = new Container();
  container.load(messagingFrontendModule);
  // ... 加载大量 *-frontend-module
  const { FrontendApplication } = require('@theia/core/lib/browser');
  // ...
  (window['theia'] = window['theia'] || {}).container = container;
  return container.get(FrontendApplication).start();
})();
```

这其实就是一个**前端启动入口**：

- 创建 InversifyJS `Container`；
- 把各个前端模块（`xxx-frontend-module`）加载进容器；
- 从容器里拿出 `FrontendApplication`，然后调用 `start()`；
- 顺手把 `container` 挂到 `window.theia.container` 上，方便调试或后续使用。

换句话说：**Theia 把所有“前端能力”都打包成一堆 InversifyJS 模块，然后通过这个入口组装成一个完整应用。**

## Container：Theia 前端世界的“装配厂”

关键几行代码：

```js
const { Container } = require('@theia/core/shared/inversify');
// ...
const container = new Container();
```

这里用的 `Container` 实际上就是 InversifyJS 提供的 IoC 容器（Theia 自己 re-export 了一份）：

- 所有前端服务、视图、贡献点（contribution）都会绑定到这个容器上；
- 模块加载（`container.load(...)`）其实就是在给容器“批量注册绑定关系”；
- 后面 `container.get(FrontendApplication)` 拿到的就是**已经注入好所有依赖**的应用对象。

可以把它想象成：**Theia 前端的所有“零件”（服务/扩展）都在这里登记，然后通过容器组装成完整 IDE。**

## `container.load(module)`：Theia 模块 = 一堆 InversifyJS 绑定

你会看到大量这样的代码：

```js
const { messagingFrontendModule } = require('@theia/core/lib/browser/messaging/messaging-frontend-module');
container.load(messagingFrontendModule);

const { frontendApplicationModule } = require('@theia/core/lib/browser/frontend-application-module');
container.load(frontendApplicationModule);

// 以及一长串 await load(container, import('...-frontend-module'));
```

这些 `xxx-frontend-module` 本质上就是 **InversifyJS 的 `ContainerModule`**：

- 每个模块内部大致长这样（伪代码）：

  ```ts
  export const someFrontendModule = new ContainerModule(bind => {
    bind(ServiceA).to(ServiceAImpl).inSingletonScope();
    bind(ServiceB).to(ServiceBImpl);
    // ... 注册一堆服务、贡献点、视图等
  });
  ```

- `container.load(someFrontendModule)` 做的事情就是执行这个模块里的 `bind(...)` 逻辑，把所有服务注册到容器里。

这就解释了为什么 Theia 可以通过“加/减模块”来裁剪功能：

- 想要某个功能 → 把对应的 `xxx-frontend-module` 加到启动脚本里；
- 想禁用某个功能 → 不 load 它，或者提供自己的替代模块覆盖绑定。

从 InversifyJS 的角度看，**这就是典型的“按模块组织绑定”的高级用法**。

## preload：在应用真正启动前做准备工作

中间有一段 `preload(container)`：

```js
async function preload(container) {
  try {
    await load(container, import('@theia/core/lib/browser/preload/preload-module'));
    await load(container, import('@theia/core/lib/browser-only/preload/frontend-only-preload-module'));
    await load(container, import('@theia/api-samples/lib/browser/api-samples-preload-module'));
    const { Preloader } = require('@theia/core/lib/browser/preload/preloader');
    const preloader = container.get(Preloader);
    await preloader.initialize();
  } catch (reason) {
    console.error('Failed to run preload scripts.', reason);
  }
}
```

这里又是同一套模式：

- 把若干 “preload 模块” 作为 ContainerModule 加载进容器；
- 从容器里拿到 `Preloader` 服务；
- 调用 `preloader.initialize()` 做一系列“启动前任务”（比如缓存、语言包加载、环境检测等）。

这段代码很好地体现了 **“所有能力都通过容器获取，而不是到处 new”** 的原则：

- `Preloader` 自己的依赖也通过 InversifyJS 注入；
- 启动脚本只负责 orchestrate（编排顺序），而不是关心细节实现。

## 最后一步：从容器拿出 `FrontendApplication` 并启动

核心收尾逻辑在这里：

```js
const { FrontendApplication } = require('@theia/core/lib/browser');
// ...
function start() {
  (window['theia'] = window['theia'] || {}).container = container;
  return container.get(FrontendApplication).start();
}
```

关键点有两个：

1. **`window.theia.container = container`**
   - 方便在浏览器控制台里调试：你可以 `theia.container.get(SomeService)` 看看容器里有什么；
   - 也方便某些全局脚本访问容器（虽然从架构洁癖角度看，这算是一种“后门”）。

2. **`container.get(FrontendApplication).start()`**
   - `FrontendApplication` 本身也是一个被注入了大量依赖的类：命令系统、布局系统、菜单、状态存储等等；
   - 调用 `start()` 时，它会去：
     - 初始化各个贡献点（Command/Keybinding/Menu/Widget 等）；
     - 创建主 Shell；
     - 渲染整个前端应用。

从依赖注入的视角看：  
**Theia 的前端应用启动，其实就是“从容器里拿出一个配置好的 FrontendApplication 实例，然后让它跑起来”。**

## 这份 index.js 给我带来的几个 InversifyJS 认知

结合前面几篇 InversifyJS 笔记，再看这份 `index.js`，我大概会有这几个“小总结”：

- **Container 是唯一的“真入口”**
  - 所有前端功能、服务、扩展都以模块的形式往容器里注册；
  - 不再到处 `new`，而是把“创建权”集中到 IoC 容器里。

- **模块（ContainerModule）是 Theia 的“功能颗粒度单位”**
  - 一个 `xxx-frontend-module` 代表一块功能（文件系统、终端、Git、插件、AI 功能等）；
  - 启动脚本里 load 哪些模块，就决定了这个 Theia 变体长成什么样。

- **前端启动脚本只是“编排者”，不是“工厂”**
  - 它只负责：创建容器 → 加载模块 → 做一点 preload → 从容器里拿 FrontendApplication 启动；
  - 至于每个模块内部具体绑定了什么服务、这些服务怎么协作，全都交给 InversifyJS + Theia 自己的扩展框架。

从学习的角度来说，这个 `index.js` 是一个非常好的“桥”：  
**一头连着 InversifyJS 容器和模块系统，一头连着 Theia 的前端应用生命周期，刚好把两边的概念串在了一起。**  
理解了它，再去看单个 `xxx-frontend-module` 里是怎么 `bind()` 各种服务，会更有方向感。**

