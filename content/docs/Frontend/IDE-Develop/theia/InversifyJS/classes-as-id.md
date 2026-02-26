---
title: "用类本身作为服务标识符：简化绑定的另一种方式"
weight: 6
---

> 前面我们一直用 `Symbol` 作为服务标识符，但 InversifyJS 也支持直接用**类本身**作为标识符。  
> 这种方式在某些场景下可以简化代码，但也有一些限制和陷阱。这篇就聊聊什么时候用类作为 ID，什么时候还是用 Symbol 更稳妥。

## 类作为标识符的基本用法

当你用类作为服务标识时，**不需要 `@inject()` 装饰器**，因为 TypeScript 的元数据反射可以自动推断构造函数参数的类型：

```typescript
import "reflect-metadata";
import { Container, injectable } from "inversify";

@injectable()
class Katana {
  public hit() {
    return "cut!";
  }
}

@injectable()
class Shuriken {
  public throw() {
    return "hit!";
  }
}

@injectable()
class Ninja {
  private _katana: Katana;
  private _shuriken: Shuriken;

  public constructor(katana: Katana, shuriken: Shuriken) {
    this._katana = katana;
    this._shuriken = shuriken;
  }

  public fight() {
    return this._katana.hit();
  }

  public sneak() {
    return this._shuriken.throw();
  }
}

const container = new Container();
container.bind<Katana>(Katana).to(Katana);
container.bind<Shuriken>(Shuriken).to(Shuriken);
container.bind<Ninja>(Ninja).to(Ninja);

const ninja = container.get<Ninja>(Ninja);
console.log(ninja.fight()); // "cut!"
```

注意几个点：

- `Ninja` 的构造函数参数**没有 `@inject()` 装饰器**，但容器依然能正确注入 `Katana` 和 `Shuriken`。
- 绑定时直接用类作为标识：`container.bind<Katana>(Katana).to(Katana)`。
- 获取时也用类：`container.get<Ninja>(Ninja)`。

## 使用 `toSelf()` 简化绑定

如果要绑定的类型是具体类（不是接口），绑定语句会显得重复：

```typescript
container.bind<Samurai>(Samurai).to(Samurai); // 重复了
```

可以用 `toSelf()` 简化：

```typescript
container.bind<Samurai>(Samurai).toSelf(); // 更简洁
```

这在绑定很多具体类时会减少重复代码。

## 已知局限性：循环依赖的问题

**类作为标识符最大的坑是循环依赖**。看这个例子：

```typescript
import "reflect-metadata";
import { Container, injectable } from "inversify";

@injectable()
class Dom {
  public domUi: DomUi;
  constructor(domUi: DomUi) {
    this.domUi = domUi;
  }
}

@injectable()
class DomUi {
  public dom: Dom;
  constructor(dom: Dom) {
    this.dom = dom;
  }
}

const container = new Container();
container.bind<Dom>(Dom).toSelf().inSingletonScope();
container.bind<DomUi>(DomUi).toSelf().inSingletonScope();

const dom = container.get<Dom>(Dom); // Error!
```

会抛出类似这样的错误：

{{< tip "warning" >}}
Error: Missing required @Inject or @multiinject annotation in: argument 0 in class Dom.
{{< /tip >}}

**为什么会报错？**

当使用类作为服务标识时，InversifyJS 依赖 TypeScript 的元数据反射来推断类型。但在循环依赖的场景下：

1. 装饰器执行时，`Dom` 和 `DomUi` 可能还没完全初始化。
2. 元数据反射可能读到 `undefined`，导致 InversifyJS 认为缺少 `@inject()` 注解。
3. 即使你手动加上 `@inject(Dom)` 或 `@inject(DomUi)`，也可能因为装饰器执行顺序问题，依然报错。

**解决方案：用 Symbol 作为标识符**

```typescript
import "reflect-metadata";
import { Container, injectable, inject } from "inversify";

const TYPES = {
  Dom: Symbol("Dom"),
  DomUi: Symbol("DomUi")
};

@injectable()
class Dom {
  public domUi: DomUi;
  constructor(@inject(TYPES.DomUi) domUi: DomUi) {
    this.domUi = domUi;
  }
}

@injectable()
class DomUi {
  public dom: Dom;
  constructor(@inject(TYPES.Dom) dom: Dom) {
    this.dom = dom;
  }
}

const container = new Container();
container.bind<Dom>(TYPES.Dom).to(Dom).inSingletonScope();
container.bind<DomUi>(TYPES.DomUi).to(DomUi).inSingletonScope();

const dom = container.get<Dom>(TYPES.Dom); // 正常工作
```

用 `Symbol` 作为标识符，即使有循环依赖也能正常工作，因为 `Symbol` 在定义时就已经是确定的值了。

## 在 Theia 里的实践建议

从我读 Theia 源码的观察来看，**Theia 几乎全部用 `Symbol` 作为服务标识符**，很少直接用类。原因包括：

1. **接口绑定更灵活**：Theia 里很多服务都是绑定到接口的（`bind<IFileService>(TYPES.FileService).to(FileService)`），这样测试时可以轻松替换实现。
2. **避免循环依赖陷阱**：大型项目里很难保证没有循环依赖，用 `Symbol` 更稳妥。
3. **类型安全更好**：`Symbol` + 泛型可以做到“编译时就知道绑定关系是否正确”。

所以我的建议是：

- **简单项目、没有循环依赖、不需要接口绑定**：可以用类作为标识符，代码更简洁。
- **复杂项目、可能有循环依赖、需要接口绑定**：还是用 `Symbol` 更稳妥，这也是 Theia 的选择。

## 小结

类作为服务标识符是 InversifyJS 提供的一个便利特性，可以简化某些场景下的代码。但要注意：

- ✅ **适合**：简单依赖、具体类绑定、没有循环依赖。
- ❌ **不适合**：循环依赖、接口绑定、需要灵活替换实现。

在 Theia 这种大型项目里，`Symbol` 作为标识符是更主流、更安全的选择。