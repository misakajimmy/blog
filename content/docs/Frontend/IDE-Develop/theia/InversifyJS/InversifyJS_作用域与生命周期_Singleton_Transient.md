---
title: "InversifyJS：作用域与生命周期（Singleton / Transient 等）"
weight: 7
---

> 这篇补上 InversifyJS 里另一个经常被忽略、但在 Theia 这种长跑型应用里非常关键的点：  
> **作用域（Scope）和生命周期（Lifetime）——一个服务是全局单例，还是每次来一个新的？会不会不小心创建太多？**

## 三种常见作用域

InversifyJS 里最常用的几个作用域是：

- **Transient（默认）**：每次 `container.get()` 都创建一个新实例。  
- **Singleton**：整个容器生命周期内只有一个实例。  
- **Request**：每个“请求”范围内共享一个实例（更多用于服务端场景，Theia 前端里不常用）。

先看一个最简单的例子：

```ts
@injectable()
class Counter {
  public value = 0;
}

container.bind(Counter).toSelf(); // 默认 transient

const c1 = container.get(Counter);
const c2 = container.get(Counter);

console.log(c1 === c2); // false
```

如果我们希望 `Counter` 在整个应用中是单例，可以这样写：

```ts
container.bind(Counter).toSelf().inSingletonScope();

const a = container.get(Counter);
const b = container.get(Counter);

console.log(a === b); // true
```

## 在 Theia 里，哪些东西应该是单例？

在 Theia 源码里，大多数“服务”（Service）都是单例，比如：

- `CommandRegistry`  
- `MenuModelRegistry`  
- `WorkspaceService`  
- `FileService`  
- 各种 `*Contribution`（命令、菜单、键位、视图贡献等）

原因很直接：

- 它们代表的是“全局状态”或“全局入口”：一个应用只需要也只应该有一个；  
- 单例可以避免状态不一致（比如两个不同的 `WorkspaceService` 各自维护一套工作区信息）。

典型绑定写法：

```ts
container.bind<WorkspaceService>(WorkspaceService).to(WorkspaceServiceImpl).inSingletonScope();
```

或者在 Theia 源码里，经常会看到类似：

```ts
bind(WorkspaceService).toSelf().inSingletonScope();
bind(WorkspaceFrontendContribution).toSelf().inSingletonScope();
bind(FrontendApplicationContribution).toService(WorkspaceFrontendContribution);
```

这里 `WorkspaceFrontendContribution` 是单例，然后通过 `toService` 暴露为某个接口的一种实现。

## 哪些场景更适合 transient（每次一个新实例）？

也有一些对象更适合 transient，比如：

- 临时的对话框 / 表单视图；  
- 某些“短生命周期”的 helper 对象；  
- 为每个请求/任务创建的 handler（前端少见，后端更多）。

在 Theia 里，很多 UI Widget 其实是由工厂创建，而不是直接从容器 `get()` 出来，这时通常会结合工厂模式使用：

```ts
export const MyWidgetFactory = Symbol('MyWidgetFactory');

export type MyWidgetFactory = () => MyWidget;

bind(MyWidget).toSelf(); // 默认 transient：每次创建新实例
bind(MyWidgetFactory).toFactory<MyWidget>(ctx => () => {
  return ctx.container.get(MyWidget);
});
```

这样每次调用 `factory()` 都会拿到一个新的 `MyWidget`，非常适合“多窗口、多视图”的场景。

## 生命周期与资源释放：和 disposable 的关系

前一篇《disposable 与组件生命周期》里已经提过：  
**作用域决定“这个对象会活多久”，而 disposable 决定“它在寿终正寝时要怎么收尾”。**

在结合使用时，几个常见点是：

- 单例服务里经常会：
  - 持有对其它服务/事件源的引用；  
  - 在 `dispose()` 里统一注销监听、取消注册、清理资源。
- transient 对象（比如临时 Widget）：
  - 生命周期通常由 Shell/Layout 控制；  
  - 在 Widget 的 `dispose()` / `onBeforeDetach` 里收尾。

Theia 里很多 binding 会同时指定作用域 + 实现 `IDisposable` 接口，这两个概念配合起来才算一个完整的“生命周期管理方案”。

## 在 Theia 源码里如何“感受”这些作用域？

如果你想在 Theia 源码里训练一下对作用域的直觉，可以尝试：

1. 找一些 `bind(...).inSingletonScope()` 的地方，看对应类型是做什么的：  
   - 通常是“核心服务/贡献点”，比如 `FrontendApplicationContribution` 的各种实现。

2. 找一些没有指定作用域、或者专门用工厂创建的类型：  
   - 通常是 Widget 或 UI 组件，生命周期交给布局系统管理。

3. 在调试时打印对象引用：  
   - 比如在多个地方注入同一个服务，`console.log(service === otherService)` 验证是不是单例。

对我来说，理解 InversifyJS 的作用域之后，再看 Theia 的绑定文件时会更有安全感：  
**知道哪些对象应该只存在一份，哪些对象应该按需创建，而不是靠“印象”猜。**

