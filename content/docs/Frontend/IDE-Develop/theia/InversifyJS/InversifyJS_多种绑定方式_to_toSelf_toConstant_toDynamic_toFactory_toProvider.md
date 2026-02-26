---
title: "InversifyJS：多种绑定方式（to / toSelf / toConstantValue / toDynamicValue / toFactory / toProvider）"
weight: 8
---

> 这一篇专门把 `bind()` 的几种常见变体过一遍：  
> **`to`、`toSelf`、`toConstantValue`、`toDynamicValue`、`toFactory`、`toProvider`**，  
> 并顺带聊聊它们在 Theia 这种长跑型应用里的典型用法。

## 1. `to`：最常见的“接口 → 实现类”绑定

这是你最常见到的形式：

```ts
container.bind<IFileService>(TYPES.FileService).to(FileServiceImpl).inSingletonScope();
```

含义很直接：

- 当有人 `@inject(TYPES.FileService)` 时，容器会 new 一个 `FileServiceImpl` 给他；  
- 结合 `inSingletonScope()` 使用，就变成“整个容器里只有一个 `FileServiceImpl` 实例”。

在 Theia 里，大多数“核心服务”的绑定都是这种形式，例如 `WorkspaceService`、`FileService`、`CommandRegistry` 等。

## 2. `toSelf`：具体类自己就是标识符

当你不需要接口 + Token 这层抽象时，可以用类本身作为标识符：

```ts
@injectable()
class WorkspaceService { /* ... */ }

container.bind(WorkspaceService).toSelf().inSingletonScope();

const svc = container.get(WorkspaceService);
```

这相当于：

```ts
container.bind<WorkspaceService>(WorkspaceService).to(WorkspaceService);
```

只是更简洁一些。  
在 Theia 里，经常会看到先用 `toSelf()` 绑定自身，再通过 `toService()` 暴露为某个接口的实现：

```ts
bind(MyContribution).toSelf().inSingletonScope();
bind(FrontendApplicationContribution).toService(MyContribution);
```

这表示：

- Container 里有一个单例的 `MyContribution`；  
- 任何地方如果注入 `FrontendApplicationContribution[]`，会拿到这个 `MyContribution` 实例作为其中一员。

## 3. `toConstantValue`：绑定常量/配置/单例对象

有些东西你并不希望容器帮你 new，而是希望直接把一个现成的值塞进容器，例如配置对象、全局常量等：

```ts
const config = {
  apiBaseUrl: 'https://api.example.com',
  featureFlags: { newUI: true },
};

container.bind(TYPES.AppConfig).toConstantValue(config);
```

之后注入：

```ts
@injectable()
class HttpClient {
  constructor(
    @inject(TYPES.AppConfig) private readonly config: AppConfig,
  ) {}
}
```

特点：

- 容器不会 new，它只是在内部存了一个引用；  
- 非常适合用来传递“只读配置”“上下文对象”等。

在 Theia 里，也有类似把 `FrontendApplicationConfig` 这样东西通过 DI 暴露出去的模式（虽然有些是通过专门的 provider 服务包装）。

## 4. `toDynamicValue`：按需、惰性、基于上下文生成实例

`toDynamicValue` 是一个非常灵活的选项：  
它允许你通过**一个函数**来生成实例，可以访问容器上下文、环境变量等。

```ts
container.bind(TYPES.Clock).toDynamicValue(() => ({
  now: () => new Date(),
}));
```

或者需要用到容器中其它服务时：

```ts
container.bind(TYPES.ServiceWithDeps).toDynamicValue(ctx => {
  const dep = ctx.container.get(TYPES.OtherService);
  return new ServiceWithDeps(dep, Date.now());
});
```

适用场景：

- 需要基于运行时环境做决定（比如 dev / prod 模式选择不同实现）；  
- 需要访问容器里的其它绑定，但又不想把这些依赖写进构造函数里（某些 legacy 场景）。

在 Theia 里，很多时候会优先选择“普通 `to()` + 构造函数注入”，  
`toDynamicValue` 更像是你在做一些**适配/桥接层**时的高级选项。

## 5. `toFactory`：返回一个“工厂函数”

当你需要一个**“按需创建实例的工厂”**时，可以用 `toFactory`：

```ts
export const TYPES = {
  WidgetFactory: Symbol('WidgetFactory'),
} as const;

@injectable()
class MyWidget { /* ... */ }

container.bind(MyWidget).toSelf(); // transient: 每次 get 都是新的

container.bind<() => MyWidget>(TYPES.WidgetFactory).toFactory(ctx => {
  return () => ctx.container.get(MyWidget);
});
```

使用：

```ts
@injectable()
class SomeContribution {
  constructor(
    @inject(TYPES.WidgetFactory)
    private readonly widgetFactory: () => MyWidget,
  ) {}

  openView() {
    const widget = this.widgetFactory();
    // 把 widget 加到布局中
  }
}
```

在 Theia 里，**很多 Widget 的创建就是通过类似的“WidgetFactory”完成的**（有时是 Theia 自己封装的 widget factory token），  
这样可以让布局系统在需要时创建新视图，而不是一开始就 new 好所有东西。

## 6. `toProvider`：支持异步创建（返回 Promise 的工厂）

`toProvider` 和 `toFactory` 类似，但更偏异步场景：  
它返回一个**异步函数**（通常是 `() => Promise<T>` 或 `(args) => Promise<T>`）。

```ts
import { Provider } from 'inversify';

export const TYPES = {
  RemoteDataProvider: Symbol('RemoteDataProvider'),
} as const;

container.bind<Provider<Data>>(TYPES.RemoteDataProvider).toProvider<Data>(ctx => {
  return async () => {
    const http = ctx.container.get(HttpClient);
    const resp = await http.get('/data');
    return resp.data as Data;
  };
});
```

使用：

```ts
@injectable()
class DataConsumer {
  constructor(
    @inject(TYPES.RemoteDataProvider)
    private readonly getData: () => Promise<Data>,
  ) {}

  async load() {
    const data = await this.getData();
    // ...
  }
}
```

适用场景：

- 需要**延迟加载**某些重资源（例如远程数据、惰性初始化组件）；  
- 需要把“获取过程”本身抽象成 DI 提供的能力，而不是在消费方硬编码 fetch 逻辑。

在 Theia 中，部分“provider 风格”的功能（例如某些异步服务获取）可以用类似思路实现，  
不过核心框架本身更多使用普通 `to()` + async 方法组合。

## 小结：选哪种绑定，取决于“对象是谁创建的”和“何时创建”

从我自己的使用体验来看，可以用一句话来记住这些绑定方式的差异：

- **`to` / `toSelf`**：  
  - “这个类由容器来 new，生命周期由作用域控制”——最常见的服务/组件绑定。

- **`toConstantValue`**：  
  - “这个值是现成的配置/单例，容器只负责转发引用”。

- **`toDynamicValue`**：  
  - “创建逻辑比较特殊，或者需要访问运行时上下文/容器本身时，用它兜一下”。

- **`toFactory` / `toProvider`**：  
  - “我需要一个能在以后某个时刻再创建实例（同步/异步）的函数”。

在 Theia 这种架构里，大部分时候你会只用到 `to` / `toSelf` + 作用域 + Symbol/接口绑定；  
但一旦遇到**工厂、异步加载、运行时选择实现**这类稍微复杂一点的需求，这些额外的绑定方式就能帮你少写很多胶水代码。

