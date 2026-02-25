---
title: "Lumino 的 disposable 与组件生命周期：东西造出来，总要有人负责善后"
weight: 8
---

> 前面在讲 Widget / signaling / messaging 的时候，其实一直有个“隐身角色”：  
> **谁来负责把事件监听、定时器、模型订阅这些东西在合适的时机清掉？**  
> 这篇就从 `@lumino/disposable` 讲起，顺便把几类常见组件的生命周期串一遍。

## `@lumino/disposable`：一个很小但到处都在用的模式

`@lumino/disposable` 暴露的大致内容很简单：

- `IDisposable` 接口：只有一个 `dispose(): void` 方法；  
- `DisposableDelegate`：包装一个 `() => void` 的小工具，`dispose()` 时调用这个函数；  
- `DisposableSet`：一组 disposable 的集合，可以一次性 `dispose()` 全部。

最朴素的用法是：

```ts
import { DisposableDelegate, DisposableSet } from '@lumino/disposable';

// 单个 disposable：包一段“清理逻辑”
const d1 = new DisposableDelegate(() => {
  console.log('clean up something');
});

// disposable 集合：方便成批管理
const bag = new DisposableSet();
bag.add(d1);
bag.add(new DisposableDelegate(() => console.log('another cleanup')));

// 在合适的时机统一释放
bag.dispose();
```

看起来非常简单，但它的价值在于——**给“资源释放”这件事一个统一的抽象**，  
在大型框架（比如 Theia）里，很多服务/组件都会实现 `IDisposable`，方便在应用关闭或容器销毁时统一清理。

## 把 disposable 和 Widget 生命周期结合起来

以 Widget 为例，一个常见套路是：

- 在 `onAfterAttach` 里订阅事件、signal、定时器等；  
- 在 `onBeforeDetach` 或 Widget 自身的 `dispose()` 里用 `DisposableSet` 把这些资源一次性释放。

一个简单的示例：

```ts
import { Widget } from '@lumino/widgets';
import { DisposableSet, DisposableDelegate } from '@lumino/disposable';
import { ISignal, Signal } from '@lumino/signaling';

class Model {
  private _changed = new Signal<Model, void>(this);

  get changed(): ISignal<Model, void> {
    return this._changed;
  }

  trigger() {
    this._changed.emit(void 0);
  }
}

class MyWidget extends Widget {
  private _model = new Model();
  private _toDispose = new DisposableSet();

  constructor() {
    super();
    this.addClass('my-DisposableDemo');
  }

  protected onAfterAttach(msg: any): void {
    // 1. 订阅 model 的 signal
    this._model.changed.connect(this.onModelChanged, this);
    this._toDispose.add(
      new DisposableDelegate(() => {
        this._model.changed.disconnect(this.onModelChanged, this);
      }),
    );

    // 2. 绑定 DOM 事件
    const handler = () => this._model.trigger();
    this.node.addEventListener('click', handler);
    this._toDispose.add(
      new DisposableDelegate(() => {
        this.node.removeEventListener('click', handler);
      }),
    );
  }

  protected onBeforeDetach(msg: any): void {
    // 3. 在离开 DOM 前统一清理
    this._toDispose.dispose();
  }

  private onModelChanged(sender: Model): void {
    console.log('model changed');
  }
}
```

这段代码表达的意思很简单：  
**只要 Widget 不再挂在页面上，它附带的各种监听/订阅都应该跟着寿终正寝**，而不是在后台默默泄露。

在 Theia 里也有类似的模式，只不过很多时候是通过依赖注入 + `DisposableCollection`（和 Lumino 很像）来管理。

## 常见组件的生命周期：从 Application 到 Widget

顺着 `disposable` 的话题，把几种常见层级的生命周期按“自上而下”捋一下（简化版，只讲和清理相关的点）：

### 1. Application 层

- **创建阶段**：  
  - new `Application` / `FrontendApplication`；  
  - 构造 `CommandRegistry`、`Shell`、菜单系统等；  
  - 注册各种服务、贡献点（在 Theia 里是 *Contribution*）。  
- **运行阶段**：  
  - 通过命令、菜单、布局等创建/销毁一批批 Widget / 视图。  
- **销毁阶段**：  
  - 应用关闭时，调用 Application 的 `dispose()`，它再递归调用 shell / 服务等的 `dispose()`。

这里 `@lumino/disposable` 的作用是给“服务”和“子系统”一个统一的离场接口，便于 Application 在 shutdown 时不遗漏。

### 2. Shell / 布局层

以包含 DockPanel 的 Shell 为例：

- **创建/attach**：DockPanel 被挂到 DOM 上，子 Widget 依次收到 `onAfterAttach`。  
- **布局变化**：拆分/合并/关闭标签，部分 Widget 被从 DockPanel 中移除，触发 `onBeforeDetach`。  
- **彻底销毁**：Shell 自身被 `dispose()`，通常会把内部所有 Widget `dispose()` 一遍。

在这个层级上，比较重要的是：**DockPanel 自己也实现了 `dispose()`，会清理内部的布局状态和监听**，  
不然长时间折腾布局可能会造成内存堆积。

### 3. 单个 Widget 层

对于一个普通 Widget，生命周期大概是这样的：

1. **构造函数**：初始化状态，但不要操作 DOM（因为还没 attach）。  
2. **`onAfterAttach`**：  
   - 可以安全地访问 `this.node` 所在的 DOM 环境；  
   - 适合绑定事件、启动定时器、订阅模型 signal。  
3. **`onUpdateRequest` / `onResize`**：  
   - 响应外界的更新/布局变化；  
   - 通常通过 `this.update()` 触发。  
4. **`onBeforeDetach`**：  
   - 从 DOM 中卸载前最后的机会；  
   - 适合解除事件监听、取消定时任务、断开 signal 订阅（通常配合 `DisposableSet`）。  
5. **`dispose()`**：  
   - Widget 生命周期的最终终点；  
   - 会确保不再接受消息循环，内部资源应在这里全部释放。

在 Theia 的 ReactWidget 等封装里，这套生命周期会再被转译成更贴近 React 的钩子，但底层还是 Lumino 的消息/生命周期模型在运转。

## 在 Theia/Lumino 里看“生命周期 + 资源释放”的几个观察点

实际翻代码或写扩展时，我自己会刻意留意这些地方：

- **有没有实现/继承某个 `IDisposable` / `DisposableCollection`**：  
  - 有的话，基本可以推断这个对象在某处会被集中 `dispose()`；  
  - 自己往里面加资源（比如 `toDispose.push(...)`）就比较安全。
- **Widget 是否在 `onBeforeDetach` 或 `dispose()` 里对事件/信号做了对称的清理**：  
  - 如果只在 `onAfterAttach` 里 `addEventListener` 或 `connect`，没有拆，就要小心可能的泄露。  
- **服务/单例对象里有没有长生命周期的订阅**：  
  - 比如 singleton service 订阅了很多 view/model 的事件，却从不释放，这种在 IDE 跑久了很容易炸内存。

把这些模式装进脑子之后，再看 Theia / Lumino 源码里各种“清理逻辑”，会觉得亲切很多：  
**大多数看起来“啰嗦”的 dispose 代码，其实都是在给长跑型应用买安全感。**

## 小结：disposable 是“看不见，但处处在”的一层保障

总结一下这一篇想说的：

- `@lumino/disposable` 本身非常小，但给“资源释放”提供了一个统一接口和组合工具；  
- 把它和 Widget / Shell / Application 等不同层级的生命周期结合起来，可以形成一套相对清晰的“谁负责善后”的约定；  
- 在像 Theia 这种长时间运行的 IDE 场景里，这种模式对避免内存泄露、事件乱飞有很现实的意义。

写这一篇更像是给自己加一个过滤器：  
**以后看到 `dispose()`、`Disposable*`、`onBeforeDetach` 这些字样时，脑子里会自动敲个钟——这里是在讲“善后”，值得多看两眼。**

