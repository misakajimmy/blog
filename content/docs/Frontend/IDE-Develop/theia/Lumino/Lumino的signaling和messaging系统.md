---
title: "Lumino 的 signaling 和 messaging：状态与生命周期的底噪"
weight: 3
---

> 这篇就把 Widget 背后的两条“暗线”单拎出来：**signaling** 和 **messaging**。  
> 一个负责「谁对谁说话」，一个负责「什么时候该说话」，很多看起来“魔法般自动”的行为，其实都是这两套机制在背后运转。

## 先分清楚：signaling vs messaging 到底谁管啥？

我自己的粗暴理解：

- **signaling**：更像是“业务层事件/信号系统”，解决的是——  
  **「A 的状态变了，B 想知道」**。  
  - 典型场景：模型变化通知视图、多个 Widget 之间同步选中状态等。
- **messaging**：更偏“框架内部的消息循环”，解决的是——  
  **「这个 Widget 现在应该执行哪个生命周期钩子」**。  
  - 典型场景：`onResize` / `onUpdateRequest` / `onAfterAttach` 等这些钩子，其实都是通过 messaging 派发进来的。

一句话：  
**signaling 是「业务信号」；messaging 是「UI 框架自己的消息泵和生命周期调度」。**

## signaling：在 Widget / 模型之间传递“我变了”的信号

先看一个 `@lumino/signaling` 的最小例子，感受一下 API 风格：

```ts
import { ISignal, Signal } from '@lumino/signaling';

// 一个简单的模型对象
class CounterModel {
  private _value = 0;
  private _changed = new Signal<CounterModel, number>(this);

  // 对外暴露只读的 signal
  get changed(): ISignal<CounterModel, number> {
    return this._changed;
  }

  get value(): number {
    return this._value;
  }

  increment(): void {
    this._value++;
    // 发出“我变了”的信号
    this._changed.emit(this._value);
  }
}

// 某个监听它的视图/Widget
const model = new CounterModel();

model.changed.connect((sender, value) => {
  console.log('Counter changed:', sender, value);
});

model.increment(); // 控制台会输出 Counter changed: ...
```

几个细节：

- `Signal<Sender, Args>` 里的第一个泛型是“信号的发送者类型”，第二个是“携带的数据类型”。  
- `connect` 接受一个回调 `(sender, args) => {}`，这点跟 Qt 的 signal/slot 有点神似。  
- Widget 通常不会自己 new 一个 Signal，而是：
  - 要么监听别的对象的信号。  
  - 要么在某个“模型对象”里管理 signal，然后 Widget 订阅它。

在 Theia 这种架构里，signaling 非常适合放在「状态/模型」那一层，让 UI 只是订阅并响应，而不是自己维护一堆 EventEmitter。

## signaling + Widget 的一个组合小例子

再来一个「模型 + Widget」的小组合，接上上一篇的 Widget 思路：

```ts
import { ISignal, Signal } from '@lumino/signaling';
import { Widget } from '@lumino/widgets';

class CounterModel {
  private _value = 0;
  private _changed = new Signal<CounterModel, number>(this);

  get changed(): ISignal<CounterModel, number> {
    return this._changed;
  }

  get value(): number {
    return this._value;
  }

  increment(): void {
    this._value++;
    this._changed.emit(this._value);
  }
}

class CounterView extends Widget {
  constructor(private model: CounterModel) {
    super();
    this.addClass('my-CounterView');

    // 订阅模型变化
    this.model.changed.connect(this.onModelChanged, this);
  }

  protected onAfterAttach(msg: any): void {
    this._render();
    this.node.addEventListener('click', () => {
      this.model.increment();
    });
  }

  private onModelChanged(sender: CounterModel, value: number): void {
    this._render();
  }

  private _render(): void {
    this.node.textContent = `Clicked: ${this.model.value}`;
  }
}
```

这里可以看到几个小模式：

- Widget 不直接修改自己的“业务状态”，而是通过 model 来改。  
- Widget 通过 `signal.connect` 订阅变化，然后在回调里调用 `_render()`。  
- 从架构味道上讲，这和常见的 MVVM / 状态管理思路是一脉相承的，只是工具换成了 Lumino 的 Signal。

## messaging：Widget 生命周期背后那条消息泵

再来看 messaging。大多数时候我们只会在 Widget 里重写这些方法：

- `onAfterAttach`  
- `onBeforeDetach`  
- `onResize`  
- `onUpdateRequest`  
- ……

但在 Lumino 内部，这些并不是直接「某处手动调用」，而是通过一个消息系统来派发：

```ts
import { Message, MessageLoop } from '@lumino/messaging';
import { Widget } from '@lumino/widgets';

class MyWidget extends Widget {
  protected onUpdateRequest(msg: Message): void {
    console.log('update request:', msg);
  }
}

const w = new MyWidget();

// 手动触发一次 update 消息
MessageLoop.sendMessage(w, Widget.Msg.UpdateRequest);
```

`MessageLoop.sendMessage` 会把一条消息送进 Widget 的处理流程里，最终调用到 `onUpdateRequest`。  
在正常使用 DockPanel / Layout 的时候，这一切都是框架帮你调度好的，你只要实现对应的钩子函数就行。

还有几个配套概念：

- `Widget.Msg.UpdateRequest` / `ResizeRequest` / `AfterAttach` 等是一些内置消息常量。  
- `MessageLoop.postMessage` 则是异步排队发送，在下一轮消息循环里处理，避免同步调用导致递归问题。

可以理解为：**messaging 是让 Widget 的生命周期更像“消息驱动”的，而不是直接函数调用。**

## signaling + messaging 在实际阅读代码时的用法区别

我自己在看 Theia / Lumino 源码时，通常会这样区分两条线：

- 看到 `Signal` / `.connect()` / `.emit()`：  
  - 把它当成“业务事件”，去想「谁在产生这个状态变化」「谁在消费它」。  
  - 这部分逻辑通常跟 UI 框架解耦，迁移到别的 UI 技术栈也可以重用。
- 看到 `onXXXRequest` / `MessageLoop` / `Widget.Msg.*`：  
  - 把它当成“UI 框架调度”，去想「这个 Widget 在什么时机会收到这些消息」。  
  - 这部分逻辑高度绑定 Lumino 的布局和渲染机制。

在调试问题时也很有用：

- 如果是「状态没同步过去」→ 大概率看 signaling 的连接和 emit。  
- 如果是「界面渲染/尺寸不对」→ 大概率看 messaging 对 `onResize` / `onUpdateRequest` 是否按预期触发。

## 小结：为什么我愿意为这俩单独写一篇？

对我个人来说，signaling 和 messaging 有点像 Lumino 这套体系里的“底噪”：

- 不刻意关注的时候，你只会看到 Widget、Panel、DockPanel 这些“可视”的部分。  
- 但真想在 Theia 里写点更贴近底层的扩展，或者想搞明白一些诡异的刷新/状态问题时，最后都会顺藤摸瓜摸到这两块。

写这一篇，更像是给自己立一个“心智坐标系”：  
**以后看到 `Signal` / `MessageLoop` 相关的调用时，脑子里能立刻知道——  
这是在搭“谁对谁说话”的桥，还是在调度“什么时候说话”的节奏。**

