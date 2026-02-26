---
title: "InversifyJS：ContainerModule 与模块化 DI（对标 Theia 前后端模块）"
weight: 10
---

> 这一篇专门补上 InversifyJS 里一个**非常贴近 Theia 架构**的概念：`ContainerModule`。  
> 如果说 `Container` 是装配厂，那 `ContainerModule` 基本就是 Theia 里各种 `*-frontend-module` / `*-backend-module` 的直接原型。

## 为什么需要 ContainerModule？单文件 bind() 太乱了

理论上你可以在一个地方把所有绑定都写完：

```ts
const container = new Container();

container.bind(FileService).to(FileServiceImpl).inSingletonScope();
container.bind(WorkspaceService).to(WorkspaceServiceImpl).inSingletonScope();
container.bind(CommandRegistry).to(CommandRegistryImpl).inSingletonScope();
// ....
```

但在一个像 Theia 这样的大项目里，这种写法很快会变成“巨大无比的 bind 清单”：  

- 不同功能（文件系统 / 工作区 / 编辑器 / 终端 / 插件系统）混在一起；  
- 想裁剪某个功能（比如去掉 Terminal）很难；  
- 扩展/子项目想“加一坨自己的绑定”也不方便。

**ContainerModule 的目的**，就是把这堆 `bind()` 拆成**按功能分组的模块**，然后在启动时按需 `load()`。

## ContainerModule：把一组绑定封成一个模块

最小示例：

```ts
import { Container, ContainerModule } from 'inversify';

const myModule = new ContainerModule(bind => {
  bind(ServiceA).to(ServiceAImpl).inSingletonScope();
  bind(ServiceB).to(ServiceBImpl);
});

const container = new Container();
container.load(myModule);
```

这里发生的事情很简单：

- `new ContainerModule(bind => { ... })`：把一组 `bind()` 演变成一个“可加载模块”；  
- `container.load(myModule)`：执行模块里的 `bind()` 函数，把对应绑定注册到容器中。

和你在 Theia 里看到的 `frontendApplicationModule` / `filesystemFrontendModule` 的角色是一样的，只不过 Theia 在自家代码里又包了一层。

## AsyncContainerModule：支持异步初始化的模块

Inversify 还提供了 `AsyncContainerModule`，适合需要**异步初始化绑定**的场景，比如：  
在绑定前要先读取配置/远程数据。

```ts
import { Container, AsyncContainerModule } from 'inversify';

const asyncModule = new AsyncContainerModule(async bind => {
  const config = await fetchConfig();
  bind(Config).toConstantValue(config);
});

const container = new Container();

await container.loadAsync(asyncModule);
```

Theia 里多数模块还是用同步模块（因为绑定逻辑本身是同步的），  
但在你自己做某些云配置/远程能力时，`AsyncContainerModule` 会很有用。

## 对标 Theia：`*-frontend-module` / `*-backend-module` 本质上就是 ContainerModule

回忆一下我们从 `index.js` 里看到的代码：

```js
const { frontendApplicationModule } =
  require('@theia/core/lib/browser/frontend-application-module');
container.load(frontendApplicationModule);

await load(container, import('@theia/editor/lib/browser/editor-frontend-module'));
await load(container, import('@theia/filesystem/lib/browser/filesystem-frontend-module'));
// ...
```

如果深入这些模块的源码（以 TypeScript 版为例），你会发现它们通常长这样（伪代码）：

```ts
import { ContainerModule } from 'inversify';

export const filesystemFrontendModule = new ContainerModule(bind => {
  bind(FileService).to(FileServiceImpl).inSingletonScope();
  bind(FileDialogService).to(FileDialogServiceImpl).inSingletonScope();
  // ... 其它与文件系统相关的绑定
});
```

也就是说：

- **每个 Theia 模块 = 一个 ContainerModule + 一组针对某个子领域的绑定**；  
- `container.load(filesystemFrontendModule)` 就是执行那组绑定，把“文件系统相关服务”挂到容器上。

这就解释了为什么 Theia 可以：

- 用“加减模块”来定制发行版（比如只保留文件浏览/编辑，不要 Git/终端等）；  
- 让第三方扩展以“模块”的形式往容器里加绑定，而不用改核心代码。

## 模块之间的依赖关系：松耦合的“功能拼图”

多个 ContainerModule 之间并不是完全独立的，比如：

- `editor-frontend-module` 可能依赖 `filesystem-frontend-module` 提供的 `FileService`；  
- `workspace-frontend-module` 可能依赖 `FileService` 和 `PreferencesService` 等。

但这种依赖**并不是硬编码的**，而是通过“服务标识符 + DI”来解耦：

- `filesystem-frontend-module`：`bind(FileService).to(FileServiceImpl)`；  
- `editor-frontend-module`：在某个 Editor 相关类上 `@inject(FileService)`；  
- 容器在加载完所有相关模块后，自然就能解析出依赖关系。

你可以把它理解为一个“功能拼图”：

- 每块拼图（模块）声明“我提供哪些服务、我需要哪些服务”；  
- DI 容器负责把整幅拼图拼起来，具体加载哪些拼图，由启动脚本/配置决定。

## 在自己项目里怎么设计模块粒度？

如果你以后在自己的项目里用 InversifyJS，可以借鉴 Theia 的模块划分方式：

- 按**业务子域**划分模块，比如：`userModule` / `projectModule` / `billingModule`；  
- 每个模块内部绑定与该子域相关的服务、视图、贡献点；  
- 在不同“发行版”或“部署形态”下加载不同组合的模块：

```ts
const container = new Container();

// 所有版本都需要的基础模块
container.load(coreModule, userModule);

// 企业版多一个 billing
if (isEnterprise) {
  container.load(billingModule);
}
```

这和 Theia 的做法几乎一模一样，只是领域不同。

## 小结：ContainerModule 是把 DI 从“单文件配置”提升到“功能级别模块”的关键

总结一下这一篇的核心点：

- `Container` 决定“谁来装配”；  
- `ContainerModule` 决定“**按什么粒度组织这堆绑定**”；  
- 在 Theia 里，这种粒度基本就是“前端功能模块 / 后端功能模块”的维度；
- 启动脚本只需要 `load()` 一堆模块，就能拼装出一个完整的 IDE。

对我个人来说，搞懂 ContainerModule 之后再看 Theia 的 `*-frontend-module` / `*-backend-module`，会有一种“哦，原来就是在写一堆 bind() 的模块化版”的释然感——  
**你可以不把它当成什么高深的“框架魔法”，就把它当成 InversifyJS 官方推荐的“按功能拆分 DI 配置”的那层语法糖。**

