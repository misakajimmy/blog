---
title: "InversifyJS：测试中的容器用法（测试容器与 rebind）"
weight: 12
---

> 这一篇聊聊 InversifyJS 在“写测试”这件事上的实际用法：  
> **怎么建一个测试专用容器、怎么用 `rebind` 替换实现、以及在像 Theia 这种重度依赖 DI 的项目里，怎样让单元测试/集成测试不那么痛苦。**

## 两种基本策略：测试容器 vs 重绑定

在测试里使用 InversifyJS，大致有两条路：

1. **单独创建一个“测试容器”**  
   - 自己 new 一个 `Container`，只注册当前测试需要的那一小撮服务；  
   - 适合“轻量级单元测试”，隔离性最好。

2. **基于现有容器，用 `rebind` 替换部分实现**  
   - 例如在 Theia 的前端容器上，把某个服务换成 fake/mock；  
   - 适合“集成测试/端到端测试”，保留大部分真实行为，只 stub 掉个别外部依赖。

下面分别展开。

## 建一个最小测试容器

单元测试最常见的做法是：  
**不要把整套应用（比如完整的 Theia 容器）搬进来，而是只搭一个自己需要的小容器。**

```ts
import 'reflect-metadata';
import { Container, injectable, inject } from 'inversify';

const TYPES = {
  Repo: Symbol('Repo'),
  Service: Symbol('Service'),
} as const;

interface Repo {
  findById(id: string): Promise<string>;
}

@injectable()
class Service {
  constructor(@inject(TYPES.Repo) private readonly repo: Repo) {}

  async getName(id: string) {
    const raw = await this.repo.findById(id);
    return raw.toUpperCase();
  }
}
```

在测试里，我们不想用真实的 Repo，而是用一个 fake：

```ts
test('Service.getName uppercases repo result', async () => {
  const container = new Container();

  const fakeRepo: Repo = {
    async findById(id: string) {
      return 'john';
    },
  };

  container.bind<Repo>(TYPES.Repo).toConstantValue(fakeRepo);
  container.bind<Service>(TYPES.Service).to(Service);

  const service = container.get<Service>(TYPES.Service);
  expect(await service.getName('1')).toBe('JOHN');
});
```

特点：

- 这个测试完全不依赖真实 Repo 的实现；  
- 替换 fake 实现非常直接（`toConstantValue` 或 `toDynamicValue` 都可以）；  
- 测试失败时更容易定位问题，因为容器里只有少量绑定。

在纯业务代码里，这是我最推荐的模式：  
**每个测试文件按需创建自己的小容器，不要直接端整个应用容器过来。**

## 在现有容器上用 `rebind` 替换实现

在像 Theia 这种框架里，有时你确实需要用到**完整容器**（带布局、命令系统等），此时可以考虑用 `rebind`：

```ts
container.rebind<SomeService>(TYPES.SomeService).to(FakeSomeService).inSingletonScope();
```

或者替换成常量值：

```ts
container.rebind<ApiClient>(TYPES.ApiClient).toConstantValue({
  async request() { /* fake impl */ },
});
```

典型流程可以是：

```ts
let container: Container;

beforeEach(() => {
  container = new Container();
  // 加载真实模块
  container.load(coreModule, workspaceModule, ...);

  // 用测试实现替换部分服务
  container.rebind<FileService>(TYPES.FileService).to(FakeFileService).inSingletonScope();
});

afterEach(() => {
  container.unbindAll();
});
```

适用场景：

- 你在测试里需要依赖 Theia 的大部分行为（命令注册、布局、菜单等）；  
- 但又不想触碰某些外部系统（真实文件系统、真实网络请求等）。

## 针对 Theia 的一些具体建议

如果你将来写 Theia 扩展并想为它写测试，我自己的偏好大概是这样分层：

- **纯业务逻辑 / 工具函数 / 小服务**  
  - 优先用“**最小测试容器**”模式；  
  - 甚至可以不引入 Inversify，直接 new + 手动注入依赖。

- **扩展中的 Contribution / Widget / Service 之间的协作**  
  - 可以建一个“**扩展级容器**”：只加载与该扩展相关的模块 + 外部少量依赖；  
  - 对不想碰的外部服务，用 `rebind` 换成 fake。

- **端到端 / 集成测试（比如通过浏览器自动化驱动 Theia）**  
  - 容器更多在应用内侧使用，你只需要**预先在某个入口脚本里 rebind** 掉一些服务；  
  - 或使用专门的测试配置/测试发行版（比如只加载部分模块）。

关键是：  
**不要在单元测试里“什么都 rebind 一遍再启动一整套 Theia”，那样调试成本很高。**

## 一些实用的小技巧

- **给 fake 实现也加上 `@injectable()`**（如果用 `to()`）：  
  - 方便在不同测试场景下复用 fake；  
  - 也可以在测试容器里用 `to(FakeImpl)` 而不是 `toConstantValue`。

- **用类型别名管理“测试版容器”**：  
  - 例如：

    ```ts
    export type TestContainer = Container;

    export function createTestContainer(): TestContainer {
      const c = new Container();
      c.load(coreTestModule, ...);
      return c;
    }
    ```

  - 方便不同测试文件共享一套基础绑定。

- **善用 `unload` / `unbindAll` 清理容器状态**：  
  - 避免“前一个测试改了绑定，后一个测试被污染”的情况。

## 小结：把容器当成“测试夹具”的一部分

从另一个角度看，InversifyJS 的容器在测试里的角色，其实很像传统测试里的“**夹具（fixture）**”：

- 测试前：搭好一套依赖环境（绑定真实实现或 fake）；  
- 测试中：获取被测对象、执行行为；  
- 测试后：清理/销毁容器，避免状态泄露。

Theia 之类框架把几乎所有服务都挂在容器上，这对测试来说反而是个优势：  
**一旦你掌握了“建测试容器 + 局部 rebind”这两招，很多看似复杂的依赖网，其实都能在测试里被驯服。**

