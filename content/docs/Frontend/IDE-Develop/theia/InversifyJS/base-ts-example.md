---
title: "InversifyJS 基础示例：从传统依赖到依赖注入"
weight: 4
---

> 这篇用一个经典的“忍者-武器”例子，展示如何从“手动 new 依赖”演进到“用 InversifyJS 做依赖注入”。  
> 这个例子虽然简单，但涵盖了 InversifyJS 最核心的概念：`@injectable()`、`@inject()`、`Container`、绑定关系。

下面是一个可运行的完整示例，你可以在 CodeSandbox 里直接编辑和运行：

<iframe src="https://codesandbox.io/embed/xenodochial-babbage-tp3us?autoresize=1&fontsize=14&hidenavigation=1&module=%2Fsrc%2Ftype.ts&theme=dark"
     style="width:100%; height:800px; border:0; border-radius: 4px; overflow:hidden;"
     title="xenodochial-babbage-tp3us"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## 代码解析：一步步理解依赖注入

### 第一步：定义服务标识符（Symbol）

```typescript
const TYPES = {
  Ninja: Symbol('Ninja'),
  Katana: Symbol('Katana'),
  Shuriken: Symbol('Shuriken')
};
```

这里定义了三个 `Symbol` 作为服务标识符。在 InversifyJS 里，**Symbol 是推荐的服务标识方式**，因为：

- 避免字符串冲突（两个不同的 Symbol 永远不会相等）。
- 在 TypeScript 里可以配合泛型做类型安全（`container.bind<Interface>(TYPES.Service).to(Implementation)`）。

在 Theia 里，你经常会看到类似的模式：

```typescript
export const FileService = Symbol('FileService');
export const CommandRegistry = Symbol('CommandRegistry');
```

### 第二步：定义业务类

```typescript
class Katana {
  hit() {
    return 'cut!';
  }
}

class Shuriken {
  throw() {
    return 'hit!';
  }
}

class Ninja {
  private _katana: Katana;
  private _shuriken: Shuriken;

  constructor(katana: Katana, shuriken: Shuriken) {
    this._katana = katana;
    this._shuriken = shuriken;
  }

  fight() {
    return this._katana.hit();
  }

  sneak() {
    return this._shuriken.throw();
  }
}
```

这是三个普通的类：

- `Katana`（武士刀）和 `Shuriken`（手里剑）是“武器”类，各自有攻击方法。
- `Ninja`（忍者）依赖这两个武器，通过构造函数注入。

**传统写法的问题**：如果不用依赖注入，你得这样创建 `Ninja`：

```typescript
const katana = new Katana();
const shuriken = new Shuriken();
const ninja = new Ninja(katana, shuriken);
```

这样写的问题：
- 每次创建 `Ninja` 都要手动 new 依赖，代码重复。
- 如果 `Katana` 或 `Shuriken` 的构造函数变了，所有创建 `Ninja` 的地方都要改。
- 测试时很难 mock 依赖（比如想测试 `Ninja` 但不想真的创建 `Katana`）。

### 第三步：标记类为可注入

```typescript
import * as inversify from 'inversify';
import 'reflect-metadata';

// 方式一：用 decorate 函数（适合不能直接用装饰器语法的场景）
inversify.decorate(inversify.injectable(), Katana);
inversify.decorate(inversify.injectable(), Shuriken);
inversify.decorate(inversify.injectable(), Ninja);
inversify.decorate(inversify.inject(TYPES.Katana), Ninja, 0);
inversify.decorate(inversify.inject(TYPES.Shuriken), Ninja, 1);
```

这里用 `decorate` 函数手动给类添加装饰器：

- `inversify.decorate(inversify.injectable(), Class)`：标记类为“可注入的”。
- `inversify.decorate(inversify.inject(Symbol), Class, index)`：标记构造函数的第 `index` 个参数需要注入哪个服务。

**更常见的写法（装饰器语法）**：

```typescript
@injectable()
class Katana {
  hit() {
    return 'cut!';
  }
}

@injectable()
class Shuriken {
  throw() {
    return 'hit!';
  }
}

@injectable()
class Ninja {
  constructor(
    @inject(TYPES.Katana) private _katana: Katana,
    @inject(TYPES.Shuriken) private _shuriken: Shuriken
  ) {}
  
  fight() {
    return this._katana.hit();
  }

  sneak() {
    return this._shuriken.throw();
  }
}
```

这种写法更简洁，也是 Theia 里最常见的模式。

### 第四步：注册绑定关系

```typescript
const container = new inversify.Container();

container.bind(TYPES.Ninja).to(Ninja);
container.bind(TYPES.Katana).to(Katana);
container.bind(TYPES.Shuriken).to(Shuriken);
```

这里创建了一个 `Container`（容器），然后把所有服务注册进去：

- `container.bind(Symbol).to(Class)`：告诉容器“当需要 `Symbol` 标识的服务时，创建 `Class` 的实例”。

### 第五步：从容器获取实例

```typescript
const ninja = container.get<Ninja>(TYPES.Ninja);
console.log(ninja.fight()); // 输出: "cut!"
console.log(ninja.sneak()); // 输出: "hit!"
```

现在不需要手动 new 了，直接 `container.get()` 就能拿到 `Ninja` 实例，容器会自动：

1. 发现 `Ninja` 需要 `Katana` 和 `Shuriken`。
2. 先创建 `Katana` 和 `Shuriken` 的实例。
3. 把这两个实例传给 `Ninja` 的构造函数。
4. 返回创建好的 `Ninja` 实例。

## 这个例子说明了什么？

从传统写法到依赖注入，最大的变化是：

- **之前**：`Ninja` 的创建者需要知道它依赖什么，手动组装。
- **现在**：`Ninja` 的创建者只需要说“给我一个 `Ninja`”，容器负责组装所有依赖。

这个模式在 Theia 里非常常见：

```typescript
// Theia 扩展里，你只需要这样写：
@injectable()
export class MyContribution implements CommandContribution {
  constructor(
    @inject(CommandRegistry) private commands: CommandRegistry,
    @inject(FileService) private fileService: FileService
  ) {}
  
  // ... 使用 commands 和 fileService
}
```

Theia 的容器会在启动时自动创建 `MyContribution` 实例，并注入它需要的 `CommandRegistry` 和 `FileService`，你不需要关心这些服务是怎么创建的、在哪里创建的。

## 下一步：更复杂的场景

这个基础例子展示了最简单的依赖注入，但实际项目里还会遇到：

- **接口绑定**：绑定到接口而不是具体类（`bind<Interface>(Symbol).to(Implementation)`）。
- **作用域**：单例（`inSingletonScope()`）vs 每次新建（`inTransientScope()`）。
- **命名绑定和标签**：同一个接口有多个实现时如何区分。
- **可选注入**：某些依赖可能不存在（`@optional()`）。

这些会在后续文章里展开。