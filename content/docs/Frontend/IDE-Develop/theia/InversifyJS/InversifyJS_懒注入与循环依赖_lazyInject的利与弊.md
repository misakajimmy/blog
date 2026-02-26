---
title: "InversifyJS：懒注入与循环依赖（lazyInject 的利与弊）"
weight: 11
---

> 前面在“类作为标识符”和“作用域”那几篇里已经提到过循环依赖，这篇就专门把这个话题拎出来，  
> **聊聊 InversifyJS 的懒注入（`lazyInject`）是怎么运作的、它在哪些情况下能帮你缓解循环依赖，以及在哪些情况下你宁可重构也别硬上。**

## 先回顾一下：循环依赖在 DI 容器里为什么麻烦？

最典型的场景：

```ts
@injectable()
class A {
  constructor(public b: B) {}
}

@injectable()
class B {
  constructor(public a: A) {}
}
```

如果直接这样绑定：

```ts
container.bind(A).toSelf().inSingletonScope();
container.bind(B).toSelf().inSingletonScope();

const a = container.get(A); // BOOM
```

问题在于：

- 创建 `A` 时，需要先创建 `B`；  
- 创建 `B` 时，又需要先创建 `A`；  
- 如果没有特殊处理，就会引发“鸡生蛋 / 蛋生鸡”的循环。

Java 里的 Spring/Guice 也都有类似问题，只是解决策略（代理对象、提前暴露引用等）略有不同。  
InversifyJS 的做法之一，就是提供一个**懒注入（lazy injection）**的机制。

## `inversify-inject-decorators` 与 `lazyInject`

`lazyInject` 不在 `inversify` 主包里，而是在 **`inversify-inject-decorators`** 这个独立包中：

```ts
import 'reflect-metadata';
import { Container, injectable } from 'inversify';
import getDecorators from 'inversify-inject-decorators';

const container = new Container();
const { lazyInject } = getDecorators(container);
```

`getDecorators(container)` 会基于当前容器返回一组装饰器，其中最常用的是 `lazyInject`：

```ts
@injectable()
class ServiceA {
  @lazyInject(ServiceB) public b!: ServiceB;
}

@injectable()
class ServiceB {
  @lazyInject(ServiceA) public a!: ServiceA;
}

container.bind(ServiceA).toSelf().inSingletonScope();
container.bind(ServiceB).toSelf().inSingletonScope();

const a = container.get(ServiceA);
console.log(a.b);      // 访问时才真正从容器里取 ServiceB
console.log(a.b.a);    // 再次访问时，ServiceB 再去取 ServiceA
```

核心区别在于：

- 普通构造函数注入：在**创建对象时**就要把所有依赖都准备好；  
- `lazyInject`：在第一次访问对应属性时，才去容器里拿依赖（“懒加载”）。

这就给打破循环提供了一个切入点：  
**A 可以先被创建出来，等真正用到 B 时，再从容器获取 B；反过来亦然。**

## 示例：用 `lazyInject` 缓解循环依赖

参考你之前在“类作为 ID”文档里的例子，我们用稍微简化的版本来说明：

```ts
import 'reflect-metadata';
import { Container, injectable } from 'inversify';
import getDecorators from 'inversify-inject-decorators';

const container = new Container();
const { lazyInject } = getDecorators(container);

@injectable()
class Dom {
  constructor(public domUi: DomUi) {}
}

@injectable()
class DomUi {
  @lazyInject(Dom) public dom!: Dom;
}

container.bind(Dom).toSelf().inSingletonScope();
container.bind(DomUi).toSelf().inSingletonScope();

const dom = container.get(Dom);
console.log(dom.domUi.dom === dom); // true
```

这里的关键是：

- `Dom` 需要 `DomUi`，通过构造函数注入；  
- `DomUi` 需要 `Dom`，但用 `@lazyInject(Dom)` 延后到属性访问时才取；  
- 容器创建 `Dom` 的过程：
  1. 创建 `DomUi` 实例（此时 `domUi.dom` 还没被访问）；  
  2. 把 `DomUi` 传给 `Dom` 构造函数；  
  3. 返回 `Dom`；  
  4. 当你访问 `dom.domUi.dom` 时，`lazyInject` 才真正去容器里拿 `Dom` 实例，并缓存起来。

这相当于在依赖图中“迟了一步”去解析某条边，从而避免了构造时的死循环。

## 懒注入不是银弹：它解决的是“访问时机”，不是“设计问题”

虽然 `lazyInject` 在一些场景下很方便，但有几个需要非常小心的点：

1. **它隐藏了依赖**  
   - 构造函数注入时，类的依赖一目了然；  
   - 用 `lazyInject` 时，依赖藏在类体内部，不看源码很难知道它依赖了谁。

2. **容易被滥用来“修补坏设计”**  
   - 真正健康的架构应该尽量避免强互相依赖（尤其是服务层）；  
   - 如果发现自己到处用 `lazyInject` 才能跑起来，很可能说明模块划分出了问题。

3. **IDE/类型推断的体验稍差一些**  
   - 属性上加 `!` 非空断言，初学者容易被绕晕；  
   - 对某些静态分析工具来说也不如构造函数注入清晰。

所以对我来说，`lazyInject` 更像是一个“**在特殊情况下用的应急工具**”，而不是日常推荐的注入方式。

## 在 Theia 这种架构里，应该怎么看待懒注入？

Theia 自己的源码里，主流还是**构造函数注入 + 接口/符号绑定**这一路：  

- 服务/贡献点之间尽量通过抽象接口解耦；  
- 把确实难以打散的“互相关联”对象，设计成由第三方管理的更高层对象（比如 Manager/Coordinator 模式），而不是让两边互相硬依赖。

如果在 Theia 扩展里考虑用 `lazyInject`，我个人会建议：

- ✅ 可以考虑用的场景：
  - 某两个类之间确实有“互相持有引用”的需求，并且已经尽量收敛在某个子模块内部；  
  - 为了避免改动太大，先用 `lazyInject` 做一个“折中方案”，再择机重构。

- ❌ 尽量避免的场景：
  - 核心服务层之间大量互相 `lazyInject`；  
  - 把 `lazyInject` 当成解决所有循环依赖的常用手段，而不是先思考模块拆分。

在大型前端/工具型应用里，一个不错的经验是：  
**如果你经常需要用懒注入，可能暗示你的依赖图有点“纠缠不清”，值得退后一步重想一下边界。**

## 小结：把 lazyInject 当成“特例工具”，而不是日常注入方式

这一篇想强调几个简单的点：

- 循环依赖在 DI 世界里是常见难题，懒注入提供了一种“通过延后访问时机来打破死循环”的技术手段；  
- `inversify-inject-decorators` 里的 `lazyInject` 可以在某些场景（尤其是 UI 组件互相持有）下派上用场；  
- 但从架构角度看，它更像是“修补缝隙的胶水”，而不是应该大量铺开的日常模式。

对我自己来说，掌握 `lazyInject` 的最大价值在于：  
**遇到循环依赖时有一个临时的安全阀可以用，同时也能提醒自己——是不是该停下来审视一下整体依赖图的设计了。**

