---
title: "InversifyJS：高级绑定（命名绑定、标签绑定、多重注入、可选注入）"
weight: 9
---

> 这篇补上几块“看文档时很容易跳过，但在复杂项目里很有用”的特性：  
> **命名绑定（whenTargetNamed）、标签绑定（whenTargetTagged）、多重注入（multiInject）、可选注入（optional）**。  
> 这些东西合在一起，基本构成了 Theia 里那种“很多实现挂在同一个接口下”的能力。

## 命名绑定：一个接口，多种实现，用名字区分

场景：  
你有一个接口 `Formatter`，但有多种实现，比如 `JsonFormatter` / `YamlFormatter`。  
希望在同一个服务标识下注册多个实现，然后在注入时按名字区分。

### 注册阶段

```ts
export const TYPES = {
  Formatter: Symbol('Formatter'),
} as const;

export interface Formatter {
  format(input: any): string;
}

@injectable()
class JsonFormatter implements Formatter {
  format(input: any) {
    return JSON.stringify(input, null, 2);
  }
}

@injectable()
class YamlFormatter implements Formatter {
  format(input: any) {
    // 伪代码
    return toYaml(input);
  }
}

container.bind<Formatter>(TYPES.Formatter).to(JsonFormatter).whenTargetNamed('json');
container.bind<Formatter>(TYPES.Formatter).to(YamlFormatter).whenTargetNamed('yaml');
```

### 注入阶段

```ts
@injectable()
class ReportService {
  constructor(
    @inject(TYPES.Formatter) @named('json') private readonly jsonFormatter: Formatter,
    @inject(TYPES.Formatter) @named('yaml') private readonly yamlFormatter: Formatter,
  ) {}
}
```

关键点：

- `whenTargetNamed('xxx')` 在 binding 端打上“名字标签”；  
- 注入时用 `@named('xxx')` 精确指定要的是哪一个实现。

在 Theia 里，类似的模式会用在某些“同一接口不同 flavor”的场景，例如不同的 debug adapter、不同的语言后端等。

## 标签绑定：用键值对来表达“这个实现的特性”

命名绑定是用“一个 name 字段”区分实现；  
标签绑定则是用“键值对标签”区分，更灵活一些。

### 注册阶段

```ts
container
  .bind<Formatter>(TYPES.Formatter)
  .to(JsonFormatter)
  .whenTargetTagged('type', 'json');

container
  .bind<Formatter>(TYPES.Formatter)
  .to(YamlFormatter)
  .whenTargetTagged('type', 'yaml');
```

### 注入阶段

```ts
@injectable()
class ReportService {
  constructor(
    @inject(TYPES.Formatter) @tagged('type', 'json') private readonly jsonFormatter: Formatter,
    @inject(TYPES.Formatter) @tagged('type', 'yaml') private readonly yamlFormatter: Formatter,
  ) {}
}
```

标签绑定的好处是：  
**可以用多个标签组合表达更丰富的条件**，比如 `('language', 'ts') + ('mode', 'strict')`。

## 多重注入：一次性拿到“这一类所有实现”

在 Theia 里，你会经常看到这种模式：  
一个扩展点有很多实现（多个 `Contribution`），启动时要把它们统统注入进来，然后遍历调用。

InversifyJS 用 `multiInject` 支持这种模式。

### 注册阶段：正常多次 bind 即可

```ts
export const TYPES = {
  Contribution: Symbol('Contribution'),
} as const;

@injectable()
class FooContribution { /* ... */ }

@injectable()
class BarContribution { /* ... */ }

container.bind(TYPES.Contribution).to(FooContribution);
container.bind(TYPES.Contribution).to(BarContribution);
```

### 注入阶段：用 `@multiInject`

```ts
import { multiInject } from 'inversify';

@injectable()
class ContributionManager {
  constructor(
    @multiInject(TYPES.Contribution)
    private readonly contributions: ReadonlyArray<unknown>, // 可以用具体接口
  ) {}

  initializeAll() {
    for (const c of this.contributions) {
      // 调用每个贡献点的方法
    }
  }
}
```

这和 Theia 里的 `FrontendApplicationContribution` / `CommandContribution` 等模式高度吻合：  
**容器里可以有很多实现，启动时统一注入成一个数组，按顺序调用。**

## 可选注入：这个依赖可能没有也没关系

有时候某个依赖不是必须的：  
例如某个功能只有在特定模块存在时才可用，否则就静默禁用。

InversifyJS 提供 `@optional()` 装饰器来表达这一点：

```ts
import { optional } from 'inversify';

@injectable()
class MaybeUseFeatureX {
  constructor(
    @inject(TYPES.FeatureX) @optional() private readonly featureX?: FeatureX,
  ) {}

  doSomething() {
    if (this.featureX) {
      this.featureX.run();
    } else {
      // 安静地退化行为
    }
  }
}
```

注意：

- 如果没有 `@optional()`，而容器里又没有对应的绑定，`container.get()` 会直接抛错；  
- 加了 `@optional()` 之后，如果找不到绑定，对应参数会是 `undefined`。

在类似插件系统、可选模块的场景里非常有用。

## 在 Theia 里的影子

虽然 Theia 自己对 InversifyJS 做了一层封装（各种 `Contribution`、`toService` 等），但这些高级绑定特性背后的抽象是一致的：

- **命名/标签绑定**：多个实现挂在同一接口下，按条件注入某一类；  
- **多重注入**：把所有实现一次性注入进来，像处理插件那样遍历调用；  
- **可选注入**：某些扩展模块存在则生效，不存在则退化。

如果你在看 Theia 源码时看到一些“按条件挑扩展”的逻辑，可以试着往 InversifyJS 这几个概念上去对照，大概率能找到一一对应的影子。

对我来说，这些“高级绑定”更像是让 IoC 容器从“简单服务注册表”升级成“插件分发中心”的一组能力——  
**一旦掌握了它们，就能在自己的工程里更自然地建出类似 Theia 那样的扩展点机制。**

