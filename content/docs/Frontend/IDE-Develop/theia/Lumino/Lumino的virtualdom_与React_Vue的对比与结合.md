---
title: "Lumino 的 virtualdom：轻量 VDOM、对比 React/Vue 以及如何结合使用"
weight: 7
---

> 前几篇基本都站在“Lumino 自己的一亩三分地”里看，这篇稍微往外看一步：  
> **Lumino 的 `@lumino/virtualdom` 是什么东西，它和 React/Vue 这种主流框架的 VDOM 有什么不一样，以及如果项目里已经用了 React/Vue，要怎么和 Lumino 玩到一起。**

## `@lumino/virtualdom` 是什么？

官方的定位很简单：**一个极轻量的 Virtual DOM 实现，主要服务于 Lumino 自己的 Widget 体系**。  

使用方式大致是：

- 用 `h()` 函数描述一棵虚拟节点树；  
- 调用 `VirtualDOM.render()` 把这棵树“打补丁”到某个真实 DOM 节点下；
- Lumino 内部的一些 Widget（比如菜单、命令面板等）都是用这套 VDOM 来写渲染逻辑的。

一个最小例子（只看基本感受）：

```ts
import { h, VirtualDOM } from '@lumino/virtualdom';

const vnode = h.div(
  { className: 'my-box' },
  h.h1('Hello Lumino virtualdom'),
  h.p('This is a simple paragraph.'),
);

// 把虚拟节点渲染到某个容器里
const host = document.getElementById('app')!;
VirtualDOM.render(vnode, host);
```

特点很明显：

- API 风格有点像 React 的 `createElement`，但借助 `h.tagName()` 简化了一些。  
- 不搞组件生命周期、状态管理、hook 之类的大系统，只负责“描述一棵树，然后高效地 patch 到 DOM 上”。

## 和 React/Vue 的对比：它刻意“不长大”的地方

我自己的理解是：**Lumino 的 virtualdom 更像是“给框架内部组件用的工具库”，而不是一个完整的 UI 框架**。  

对比 React/Vue，一些明显的差异：

- **无组件状态/生命周期抽象**  
  - React 有 `setState` / hooks、类组件生命周期；  
  - Vue 有响应式数据系统、`watch`、`computed` 等；  
  - Lumino virtualdom 完全不管“数据从哪来、什么时候更新”，它只管接收一棵 VNode 树，然后渲染/更新。

- **无路由 / 全家桶生态**  
  - React/Vue 的生态里会自然长出 Router、Store（Redux/Vuex/Pinia）、Form 等各种东西；  
  - Lumino virtualdom 刻意保持“只做 VDOM 打补丁”的小工具姿态——状态、路由、数据流全交给外面的世界（比如 Widget / 应用）。

- **和 Lumino Widget 深度绑定，而不是面向浏览器全局**  
  - 在很多 Lumino Widget 里，渲染代码是 `VirtualDOM.render(this.render(), this.node)` 这样的调用；  
  - 即：**Widget 负责生命周期和状态，virtualdom 只是帮它把 “render() 返回的描述” 变成 DOM**。

从阅读源码的角度看，这种设计有一个好处：  
**你在 Widget 里还是用 imperative（命令式）逻辑管理状态、调用 `update()`，只是把 DOM 细节交给 VDOM 去算 diff。**

## 在 Widget 里使用 virtualdom：一个完整小例子

结合前面 Widget 的生命周期，可以写一个“小型 React 风味”的 Widget：

```ts
import { Widget } from '@lumino/widgets';
import { h, VirtualDOM } from '@lumino/virtualdom';

class CounterWidget extends Widget {
  private _count = 0;

  constructor() {
    super();
    this.addClass('my-VdomCounter');
  }

  protected onAfterAttach(msg: any): void {
    this._render();
    this.node.addEventListener('click', this);
  }

  protected onBeforeDetach(msg: any): void {
    this.node.removeEventListener('click', this);
  }

  handleEvent(event: Event): void {
    if (event.type === 'click') {
      this._count++;
      this.update(); // 会触发 onUpdateRequest
    }
  }

  protected onUpdateRequest(msg: any): void {
    this._render();
  }

  private _render(): void {
    const vnode = h.div(
      { className: 'counter-root' },
      h.h1(`Count: ${this._count}`),
      h.p('点击任意位置增加计数'),
    );

    VirtualDOM.render(vnode, this.node);
  }
}
```

这里你可以看到：

- 没有 React 的 `useState` / `setState`，状态就是类字段 `_count`。  
- 更新时手动调用 `this.update()`，然后在 `onUpdateRequest` 里重新走一遍 `_render()`。  
- `_render()` 里完全用 VDOM 来描述 DOM 结构，VirtualDOM 负责做 diff + patch。

这对于已经习惯 React/Vue 的人来说，既有点熟悉，又保留了 Lumino 自己的“Widget 主导一切”的感觉。

## 如果工程里已经有 React/Vue，要怎么和 Lumino 配合？

现实世界里更常见的情况是：**项目主 UI 框架已经是 React/Vue，但希望用 Theia/Lumino 提供的 IDE 壳能力**，或者反过来在 Lumino/Theia 里嵌入一块 React/Vue 视图。

我目前比较认可的几种组合方式：

### 1. Lumino 作为“外壳”，React/Vue 作为内部视图

思路：

- 用 Lumino 的 Widget / DockPanel / command / 菜单等搭出外壳；  
- 写一个“桥接 Widget”，在它的 DOM 节点内挂载 React/Vue 组件。

伪代码示例（React 版）：

```ts
// Lumino 侧 Widget
import { Widget } from '@lumino/widgets';
import React from 'react';
import { createRoot, Root } from 'react-dom/client';
import { MyReactPanel } from './MyReactPanel';

class ReactHostWidget extends Widget {
  private _root: Root | null = null;

  constructor() {
    super();
    this.addClass('my-ReactHostWidget');
  }

  protected onAfterAttach(msg: any): void {
    this._root = createRoot(this.node);
    this._root.render(<MyReactPanel />);
  }

  protected onBeforeDetach(msg: any): void {
    this._root?.unmount();
    this._root = null;
  }
}
```

关键点：

- **在 Widget 的生命周期里挂/卸 React/Vue 应用**，这样不会和 Lumino 的 attach/detach 打架。  
- 这时你基本不会再用 `@lumino/virtualdom`，而是让 React 自己管理这块子树。

Theia 社区里已经有不少扩展就是这么做的：  
核心壳用 Lumino/Theia，业务视图用 React/Vue，这样既能享受 IDE 级的布局和命令系统，又能复用已有的组件库和生态。

### 2. React/Vue 作为主 UI，Lumino 组件嵌进去（不太常见但可以）

反过来的情况也存在：你的应用主架子是 React/Vue，只想要一块 Lumino 的 DockPanel / datagrid。

典型做法：

- 在 React/Vue 组件里，用一个 `ref` 拿到 DOM 容器；  
- 在 `onMounted` / `useEffect` 里创建 Lumino Widget，并用 `Widget.attach` 挂进去；
- 在组件卸载时调用相应的 dispose / detach。

伪代码就不展开了，思路和上一节是镜像关系——只是这回由 React/Vue 管生命周期，Lumino 当 guest。

### 3. virtualdom 自己 + React/Vue 混用？不推荐硬混

理论上你可以在某个 Widget 里一会儿用 `VirtualDOM.render`，一会儿又在子节点里挂 React/Vue，  
只要边界划清楚（谁管哪棵 DOM 子树）就不会直接冲突。

但从工程实践角度看，我更建议：

- **要么这一块完全交给 React/Vue**（Widget 只当壳）；  
- **要么这一块完全用 Lumino virtualdom**；  
- 避免在同一小块 UI 里又 VDOM 又 React/Vue，调试起来很烧脑。

## 和 React/Vue 相比，什么时候更适合用 Lumino 自己的 virtualdom？

结合这几篇的上下文，我自己的结论是：

- 如果这块 UI **高度和 Lumino Widget / DockPanel / command 系统一体化**，比如：  
  - 菜单、命令面板、内置对话框；  
  - 和布局、焦点、命令密切耦合的小组件；  
  那直接用 Lumino virtualdom 是最自然的选择。

- 如果这块 UI **更像一个独立业务模块**，比如：  
  - 复杂表单、可视化、业务面板；  
  - 已经有一整套 React/Vue 组件可以直接拿来用；  
  那就让 React/Vue 当里面的“小世界”，Lumino 当外壳就好。

从学习角度看，搞明白 Lumino virtualdom 的价值在于：

- 再读 Lumino 和 Theia 一些“看起来像 React，又不是 React”的组件实现时，脑子里会有个更清晰的模型；  
- 真要在 IDE 壳里嵌 React/Vue，也知道边界应该画在哪儿、哪些该交给谁来管。

