---
title: "InversifyJS：依赖注入的前世今生（从工厂模式到 Theia）"
weight: 1
---

> 这一篇算是 InversifyJS 系列的“序章”：不直接从 API 开始，而是  
> **先聊聊“没有 DI 容器的时候大家是怎么写代码的”、工厂模式是怎么一步步演化成依赖注入的，以及在 Java 生态里这些东西怎么玩，最后再回到为什么 Theia 要在前端世界选 InversifyJS。**

## 一切从“直接 new 依赖”开始

最原始的写法，大概是这样：

```ts
class OrderRepository {
  findById(id: string) { /* ... */ }
}

class OrderService {
  private repo = new OrderRepository();

  getOrder(id: string) {
    return this.repo.findById(id);
  }
}
```

问题很明显：

- `OrderService` **紧耦合** 到 `OrderRepository` 的具体实现；  
- 想换成 `CachedOrderRepository`、`RemoteOrderRepository`，要改类内部代码；  
- 单元测试时很难注入一个假的 Repository（mock / stub）。

这在小项目里还能忍，一旦项目大了，就会到处是“想改一个依赖，要改一片调用方”的情况。

## 手工构造 + 工厂模式：第一步解耦

第一步通常是**把构造权从类内部挪到外面**：

```ts
class OrderService {
  constructor(private repo: OrderRepository) {}
}

// 由外部代码负责 new 和组装
const repo = new OrderRepository();
const service = new OrderService(repo);
```

好处：

- `OrderService` 只依赖抽象（构造参数类型），不关心具体实现；  
- 测试时可以传入 `FakeOrderRepository`。

再往前走一点，就是**工厂模式（Factory）**：

```ts
class ServiceFactory {
  createOrderService(): OrderService {
    const repo = new OrderRepository();
    return new OrderService(repo);
  }
}

const factory = new ServiceFactory();
const service = factory.createOrderService();
```

工厂模式解决了“把组装逻辑集中起来”的问题，但新的问题是：

- 复杂系统里工厂会变成“超级构造函数”，里面写满各种 if/配置分支；  
- 多层依赖关系（A 依赖 B，B 依赖 C）时，工厂代码很快就变成一团。

## Java 世界的答案：Spring / Guice / CDI 等 DI 容器

在 Java 生态里，随着应用变复杂（尤其是企业应用），大家基本达成共识：  
**光靠手写工厂已经不够了，需要一个“通用的对象装配框架”来管依赖关系。**

几条主线大概是：

- **Spring（最出圈的那个）**  
  - 通过 XML/注解/Java Config 声明 Bean 以及它们的依赖；  
  - IoC 容器负责扫描、实例化、注入、生命周期管理；  
  - 开发者主要关注“有哪些 Bean/Service，依赖什么”，而不是“怎么 new”。  

- **Guice（Google 出的轻量 DI 框架）**  
  - 使用 Java 注解（`@Inject` 等）+ 模块配置类注册绑定；  
  - 更偏代码配置，少一点 XML/约定魔法。  

两者虽然风格不同，但核心理念是一致的：

1. 你**声明**“某个抽象（接口/类）对应哪个实现”；  
2. 容器帮你**自动 new 出带好依赖的对象**（构造函数注入、字段注入等）；  
3. 生命周期 / 作用域统一交给容器管理。

这套模式把“对象的创建与组装”当成一个独立关心点（Inversion of Control），  
开发者不再到处 `new`，而是**声明依赖，让容器来“喂”你需要的东西**。

## 回到 JavaScript/TypeScript：为什么也需要 DI？

很多前端/Node 项目一开始会觉得：  
“JS 这么灵活，我直接 `import` + 函数就能搞定，为什么还要 DI 容器？”

但一旦场景变成类似 Theia / VS Code / JupyterLab 这种：

- 前后端分层、服务众多；  
- 扩展点/插件系统；  
- 需要在不同实现之间切换（浏览器版 / Electron 版 / Node 版）；  
- 需要单元测试、集成测试时 mock 掉很多依赖；

你会发现：

- 直接 `import` + `new` 会让模块之间耦合很紧；  
- 想在不同 runtime 下切换实现（比如 Browser FS vs Node FS）会很痛苦；  
- 写插件/扩展的人很难“往系统里塞一个实现”而不破坏现有代码。

这时，一个像 InversifyJS 这样的 DI 容器就开始变得有价值。

## InversifyJS：把 Java 世界那套“DI 思维”搬到 TS 上

InversifyJS 做的事情本质上很像“小号的 Spring/Guice，但针对 TypeScript/JS 生态做了适配”：

- 用 TypeScript 装饰器（`@injectable`、`@inject`）+ `reflect-metadata` 提供类型信息；  
- 用 `Container` 维护“服务标识符 → 实现”的绑定表；  
- 支持作用域（单例/瞬时）、命名/标签绑定、多重注入等高级特性。

它解决的几个关键痛点和 Java 世界如出一辙：

- **对象组装逻辑集中管理**：不再到处 `new`，而是统一在模块中 `bind()`；  
- **解耦实现与消费方**：消费方只依赖抽象（接口/Token），容器背后替换实现；  
- **支持插件/扩展点**：多重绑定 + 多重注入，天然适合“贡献点”模式（Contribution）；  
- **测试友好**：可以在测试容器里重绑定（`rebind`）某些服务，用 fake/mock 实现替换。

## 为什么 Theia 这么重度使用 InversifyJS？

结合前面的 Theia / Lumino 笔记，我自己对 Theia 选择 InversifyJS 的理解大概是：

1. **Theia 本质上是一个“可扩展平台”，不是一个封闭应用**  
   - 大量功能通过 extension/module 提供；  
   - 这些扩展需要声明它们依赖什么核心服务，而不是硬编码到具体实现。

2. **前端/后端都有大量服务需要统一管理生命周期**  
   - 文件系统、工作区、编辑器管理、命令系统、布局、消息、日志……  
   - 用一个 DI 容器把这些服务的创建/生命周期集中起来，是最自然的选择。

3. **要支持不同运行时/部署形态**  
   - 浏览器 only / Electron / 后端语言服务器等；  
   - DI 容器让 Theia 可以在不同 runtime 下绑定不同实现，而扩展代码几乎不用改。

4. **希望把 Java 世界成熟的“扩展点 + DI”模式直接搬到前端**  
   - Eclipse 平台本身就有一套非常成熟的扩展/服务架构；  
   - InversifyJS 很好地在 TS 世界里复刻了类似的 DI 能力。

从这个角度看，Theia 用 InversifyJS 并不是“为了用而用”，而是：  
**为了实现“平台 + 扩展点 + 多 runtime”这一整套目标，DI 容器几乎是刚需，而 InversifyJS 是当下 TS 生态里最成熟、最契合这一诉求的选择之一。**

## 小结：先有“为什么需要 DI”，才有“选 InversifyJS”

这一篇想强调的是顺序问题：

- 不是“我想用 InversifyJS，于是需要 DI”；  
- 而是“我想做一个类似 Theia 这样的扩展型平台 → 我需要解耦依赖、集中组装、好测试 → DI 是合理路径 → 在 TS 生态里，InversifyJS 是一个合适的实现。”

对我个人来说，先把这个“前世今生”理清楚，再回头看 InversifyJS 的各种 API、Theia 的各种绑定文件时，心里会更踏实：  
**我们不是在学一个库，而是在把一整套成熟的架构思路搬到前端项目里。**

