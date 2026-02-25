---
title: "Lumino 的 Widget 系统：从概念到源码入口"
weight: 2
---

> 延续前一篇“前世今生”的笔记，这篇就专门盯着 Lumino 里最核心的那块：**Widget 系统**。  
> 目标不是翻译 API 文档，而是给自己理一套「看源码时脑子里应该有的模型」，顺手放点小 demo，方便之后查阅。

## Widget 在 Lumino 里的角色

在 Lumino 里，**Widget 基本可以当成“一切可见 UI 的最小单位”**：

- 每个 Widget 都对应一个 DOM 元素（默认是 `div`，也可以自定义）。
- Widget 负责自己的一些生命周期：什么时候挂到 DOM 上、什么时候从 DOM 上卸载、什么时候需要重绘。
- 更高级的组件（面板、停靠布局、TabBar 等）本质上也是 Widget，只是它们更偏“容器”。

所以，从 Theia 的角度看：  
**Theia 在做布局时，其实是在安排一堆 Lumino Widget 怎么排队站好；而不是直接操作 DOM。**

## 一个最小可跑的 Lumino Widget Demo

先看个最小可运行的 Widget 示例，大致感受一下写法：

```ts
import { Widget } from '@lumino/widgets';

// 1. 定义一个最简单的 Widget
class HelloWidget extends Widget {
  constructor() {
    super();
    this.addClass('my-HelloWidget');
    this.node.textContent = 'Hello from Lumino Widget 👋';
  }
}

// 2. 创建实例并挂载到页面
const widget = new HelloWidget();

// shell / DockPanel 之类的容器里，通常会有类似的 attach 逻辑
Widget.attach(widget, document.body);
```

几个点：

- `Widget` 自带一个 `node` 属性，代表它管理的 DOM 元素（默认 `div`）。
- 一般不会直接 `appendChild` 到 `document.body`，而是交给更高级的 Panel/应用壳来管理；这里为了示例直接挂载。

## Widget 的生命周期：attach/detach/resize/update

Lumino 给 Widget 设计了一套相对明确的生命周期钩子，这在复杂布局里非常重要。  
典型的几个方法：

- `onAfterAttach(msg)`：Widget 被插入到 DOM 后触发，适合做首次渲染、事件绑定。
- `onBeforeDetach(msg)`：从 DOM 中移除前触发，可以在这里清理事件、定时器等。
- `onResize(msg)`：容器大小变化时触发，用来响应布局变化。
- `onUpdateRequest(msg)`：需要重新渲染时触发（手动 `this.update()` 会发起这个请求）。

一个稍微完整一点的例子：

```ts
import { Widget } from '@lumino/widgets';

class CounterWidget extends Widget {
  private _count = 0;

  constructor() {
    super();
    this.addClass('my-CounterWidget');
  }

  // 第一次挂载时渲染 DOM
  protected onAfterAttach(msg: any): void {
    this._render();
    this.node.addEventListener('click', this);
  }

  // 卸载前清理事件
  protected onBeforeDetach(msg: any): void {
    this.node.removeEventListener('click', this);
  }

  // 简单的事件代理
  handleEvent(event: Event): void {
    switch (event.type) {
      case 'click':
        this._count++;
        this.update(); // 触发 onUpdateRequest
        break;
    }
  }

  protected onUpdateRequest(msg: any): void {
    this._render();
  }

  private _render() {
    this.node.textContent = `Clicked: ${this._count}`;
  }
}
```

这段代码有点“手工组件化”的味道：没有引入 React/Vue，而是直接围绕 Widget 生命周期和 `update()` 来组织逻辑。  
在 Theia 里，很多比较底层的 UI 扩展也是这种风格，只不过被包了一层框架自己的抽象。

## Widget 与容器：Panel / DockPanel / TabBar 的关系

Widget 自己只是一个“砖块”，真正决定布局的是各种“砖墙”——Panel 系列组件：

- **`Panel`**：最基础的容器 Widget，可以放一串子 Widget，按一个 layout（例如垂直/水平）来排布。
- **`DockPanel`**：支持 dock / split / tab 的高级容器，是 IDE 风格布局的核心。
- **`TabBar`**：显示 tab 页签的组件，通常和 `DockPanel` 或 `TabPanel` 一起使用。

粗暴一点的伪代码示意：

```ts
import { DockPanel, Widget } from '@lumino/widgets';

const dock = new DockPanel();

const w1 = new Widget();
const w2 = new Widget();
const w3 = new Widget();

w1.node.textContent = '左边的视图';
w2.node.textContent = '右上视图';
w3.node.textContent = '右下视图';

dock.addWidget(w1, { mode: 'split-left' });
dock.addWidget(w2, { mode: 'split-right' });
dock.addWidget(w3, { ref: w2, mode: 'split-bottom' });

Widget.attach(dock, document.body);
```

Theia 的 shell 其实就是在做类似的事情：  
先有一个包着 DockPanel 的根 Widget，然后把各个视图（文件树、编辑器、终端等）当作 Widget 塞进不同的区域。

## Widget 与信号/命令：更大一层的协作

单个 Widget 只是“会自己长大的一块砖”，真正构成应用的是：

- 使用 `@lumino/signaling` 在 Widget 之间传递状态变化（类似轻量版事件总线）。
- 使用 `@lumino/commands` 注册命令，然后在 Widget 里触发或响应这些命令。

一个很常见的模式是：

```ts
import { CommandRegistry } from '@lumino/commands';

const commands = new CommandRegistry();

commands.addCommand('app:hello', {
  label: 'Say Hello',
  execute: () => {
    console.log('Hello from command!');
  },
});

// Widget 里可以通过 commands.execute('app:hello') 来触发
```

在 Theia 里，会有自己一套 command/菜单/快捷键系统，但和 Lumino 的思路高度一致：  
**先把行为抽象成命令，再决定从哪儿被触发**（菜单、按钮、快捷键、命令面板……）。

## 写在最后：为什么先啃 Widget 再看 Theia？

对我个人的阅读路线来说，先啃 Lumino 的 Widget 系统有几个好处：

- 再回头看 Theia 的 shell 和 layout 相关代码时，不会把“应用层逻辑”和“布局框架逻辑”混在一起。
- 当需要在 Theia 里塞一个自定义视图时，可以更有把握地选：  
  - 是做成 React 组件包在一个 Widget 里，  
  - 还是直接写一个 Lumino Widget，更贴近底层。
- 理解 Widget 的生命周期之后，碰到“为什么这个视图 resize 不对劲 / 渲染时机不对”的问题，也有地方可以下手 debug。

后面如果有精力，我可能会专门搞一篇“**从一个简单的 Theia 扩展开始，顺藤摸瓜走到 Lumino Widget**”的实战笔记，把两边的调用链串起来；这一篇就先当 Widget 层的查阅手册了。

