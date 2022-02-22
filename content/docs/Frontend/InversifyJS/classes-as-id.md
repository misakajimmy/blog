---
title: "对类的支持"
weight: 12
---

# 对类的支持

InversifyJS 允许类对其他类的直接依赖。这样做时，您需要使用装饰器 `@injectable`，但不需要使用装饰器 `@inject`。

使用类作为服务标识时不需要 `@inject` 装饰器。因为编译器能为我们生成的元数据。但是别忘记下面的配置：

- 导入 `reflect-metadata`
- 在 `tsconfig.json` 文件中设置 `emitDecoratorMetadata` 为 true

<!-- TODO code not run-->
```Typescript
import { Container, injectable, inject } from "inversify";

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
class Ninja implements Ninja {

    private _katana: Katana;
    private _shuriken: Shuriken;

    public constructor(katana: Katana, shuriken: Shuriken) {
        this._katana = katana;
        this._shuriken = shuriken;
    }

    public fight() { return this._katana.hit(); };
    public sneak() { return this._shuriken.throw(); };

}

var container = new Container();
container.bind<Ninja>(Ninja).to(Ninja);
container.bind<Katana>(Katana).to(Katana);
container.bind<Shuriken>(Shuriken).to(Shuriken);
```

## 具体类型绑定自身

如果要解析的类型是具体类型，绑定注册会感到重复和冗长：

```Typescript
container.bind<Samurai>(Samurai).to(Samurai);
```

更好的解决方式是使用 `toSelf` 方法:

```Typescript
container.bind<Samurai>(Samurai).toSelf();
```

## 已知局限性: 类作为标识符和循环依赖

如果在循环依赖中使用类作为标识符会被抛出异常：

{{< tip "warning" >}}
Error: Missing required @Inject or @multiinject annotation in: argument 0 in class Dom.
{{< /tip >}}

例子:

```Typescript
import "reflect-metadata";
import { Container, injectable } from "inversify";
import getDecorators from "inversify-inject-decorators";

let container = new Container();
let { lazyInject } = getDecorators(container);

@injectable()
class Dom {
    public domUi: DomUi;
    constructor (domUi: DomUi) {
        this.domUi = domUi;
    }
}

@injectable()
class DomUi {
    @lazyInject(Dom) public dom: Dom;
}

@injectable()
class Test {
    constructor(dom: Dom) {
        console.log(dom);
    }
}

container.bind<Dom>(Dom).toSelf().inSingletonScope();
container.bind<DomUi>(DomUi).toSelf().inSingletonScope();
const dom = container.resolve(Test); // Error!
```

这个错误可能有点误导，因为当使用类作为服务标识，`@inject` 等注解不需要使用，如果我们添加 `@inject(Dom)` 或 `@inject(DomUi)` 的注解，依然会抛出相同的异常。因为装饰器被调用的时候，类还没有被声明，所以装饰器被调用为 `@inject(undefined)`，导致 InversifyJS 认为对应的注解没有被添加。

解决办法是使用 `Symbol`，比如 `Symbol.for("Dom")` 作为服务标识而不是 Dom 这样的类 :