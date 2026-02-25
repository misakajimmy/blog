---
title: "Lumino 的布局系统：DockPanel / SplitPanel / TabBar 一条龙"
weight: 4
---

> 前面两篇分别把 Widget 本身和 signaling/messaging 这两条“底噪”线理了一遍，这篇就顺着往上走一层：  
> **DockPanel / SplitPanel / TabBar 这一整套布局 Widget，怎么在页面上拼出一个 IDE 风格的桌面感布局。**

## 布局的基本思路：一堆容器 Widget 叠罗汉

用一句话先概括 Lumino 的布局观：

- 普通 Widget：负责“内容”。
- 各种 Panel / Bar：负责“怎么把这些内容排布在一个二维平面上”。

典型的几类：

- **`Panel`**：最基础的容器，内部可以挂多个子 Widget。
- **`BoxPanel` / `SplitPanel`**：负责水平/垂直方向的分割和拉伸。
- **`TabBar` + `TabPanel`**：提供多标签切换的 UI。
- **`DockPanel`**：把分屏 + 停靠 + Tab 组合在一起，是 IDE 布局的核心。

在 Theia 这类 IDE 里面，最外层的 Shell 一般会持有一个根 `DockPanel`，然后根据区域（left/right/bottom/main）往里面塞不同 Widget。

## SplitPanel：最基础的“可拖动分隔条”

先从相对简单的 `SplitPanel` 说起，它提供了一条可以拖动的分隔线，把空间按比例切给多个子 Widget。

```ts
import { SplitPanel, Widget } from '@lumino/widgets';

// 创建几个简单的内容 Widget
function createContent(label: string, color: string): Widget {
  const w = new Widget();
  w.addClass('my-SplitPanel-Item');
  w.node.textContent = label;
  w.node.style.background = color;
  return w;
}

const left = createContent('Left Pane', '#f5d0c5');
const right = createContent('Right Pane', '#c5d0f5');

// 创建一个水平 SplitPanel
const split = new SplitPanel({ orientation: 'horizontal' });
split.addWidget(left);
split.addWidget(right);

SplitPanel.setStretch(left, 1);  // 左右各 1 份
SplitPanel.setStretch(right, 1);

Widget.attach(split, document.body);
```

几个要点：

- `orientation` 决定分割方向：`'horizontal'` 或 `'vertical'`。  
- `setStretch` 可以简单控制每个子 Widget 占多大比例。  
- 用户拖动中间的分隔条时，内部会通过 messaging 触发子 Widget 的 `onResize`，从而驱动重绘。

在 IDE 里，`SplitPanel` 更像是很底层的一块积木，`DockPanel` 其实就是在更复杂的场景下组合和管理这些分裂出来的区域。

## TabBar / TabPanel：一组 Widget 共用一个“框”，靠 Tab 切换

第二块积木是 Tab，一组 Widget 共享一个可见区域，通过标签切换：

```ts
import { TabPanel, Widget } from '@lumino/widgets';

function createTab(label: string): Widget {
  const w = new Widget();
  w.addClass('my-TabPanel-Item');
  w.node.textContent = label;
  return w;
}

const panel = new TabPanel();
panel.addWidget(createTab('Tab A'));
panel.addWidget(createTab('Tab B'));
panel.addWidget(createTab('Tab C'));

Widget.attach(panel, document.body);
panel.currentIndex = 0; // 默认选中第一个
```

这里 `TabPanel` 内部会管理一个 `TabBar`，以及与之对应的内容区域：

- `TabBar` 负责上面那一排可点击的标签。  
- 内容区域负责展示当前激活的那个 Widget。  

在 IDE 布局里，这个模式最典型的应用就是“一个编辑区里打开多个文件”的场景：每个编辑器是一个 Widget，中间那排文件名就是 TabBar。

## DockPanel：把 Split + Tab 全打包了

真正让布局变得“桌面感爆表”的是 `DockPanel`：它把分屏（split）、停靠（dock）和 Tab 三种能力揉到了一起。

一个最小可感受效果的例子：

```ts
import { DockPanel, Widget } from '@lumino/widgets';

function createDockWidget(label: string, color: string): Widget {
  const w = new Widget();
  w.addClass('my-DockPanel-Item');
  w.node.textContent = label;
  w.node.style.background = color;
  return w;
}

const dock = new DockPanel();

const w1 = createDockWidget('Main', '#fef3c7');
const w2 = createDockWidget('Right-Top', '#e0f2fe');
const w3 = createDockWidget('Right-Bottom', '#dcfce7');

// 第一个 Widget 作为初始内容
dock.addWidget(w1);

// 在右侧拆出一个区域
dock.addWidget(w2, { mode: 'split-right', ref: w1 });

// 在右侧区域再次向下拆分
dock.addWidget(w3, { mode: 'split-bottom', ref: w2 });

Widget.attach(dock, document.body);
```

这里的关键是第二个参数 `options`：

- `mode` 决定“怎么插入”：  
  - `'split-right'` / `'split-left'` / `'split-top'` / `'split-bottom'`：在对应方向拆出一个新区域。  
  - `'tab-after'` / `'tab-before'`：在同一区域新建一个 Tab。  
- `ref` 指定一个“参考 Widget”，表示你要相对谁进行拆分/停靠。

在 Theia 里，当你从视图菜单里打开某个面板（比如 Outline / Problems），或者把终端拖到底部区域时，背后基本都是对 `DockPanel.addWidget` 的各种组合调用。

## 布局数据的持久化和恢复（IDE 必备套路）

IDE 类应用一个很常见的需求是：**记住用户怎么折腾布局的**，下次打开时还原。  
Lumino 为此提供了布局序列化的能力——可以把当前 DockPanel 的结构 dump 成一个 JSON，然后再 restore 回来。

概念上大概是这样（简化伪代码）：

```ts
import { DockPanel } from '@lumino/widgets';

const dock = new DockPanel();

// ... 中间用户各种拖拽、拆分、合并 ...

// 1. 导出布局（可以存到 localStorage 或服务器）
const layout = dock.saveLayout();
localStorage.setItem('layout', JSON.stringify(layout));

// 2. 重新创建 DockPanel 时恢复布局
const saved = localStorage.getItem('layout');
if (saved) {
  const layoutObj = JSON.parse(saved);
  dock.restoreLayout(layoutObj);
}
```

Theia 自己在应用层还会加一层封装：  
不只是还原「长得怎样」，还要确保对应的 Widget 能重新创建出来并填充到正确位置，这里就会涉及到 ViewRegistry / WidgetFactory 之类的机制——这一块更偏 Theia 自身架构，可以等讲到 Theia Shell 时再展开。

## 这些布局 Widget 在 Theia 里的大致落位

结合前两篇，再把这几个 Widget 放回 Theia 的语境里看一下（简化心智模型）：

- **Lumino 的 Widget / DockPanel / TabBar / SplitPanel**：  
  - 负责“物理布局”和“窗口行为”——桌面感的来源。  
  - Theia 更多是作为“使用者”和“组织者”，去告诉 Lumino 怎么摆这些砖。
- **Theia 自己的 Shell / View / Contribution 层**：  
  - 决定“有哪些区域”“每个区域里塞什么 Widget”“这些视图怎么注册/销毁/还原”。  
  - 在合适的时机调用 `dock.addWidget` / `restoreLayout` 等接口。

所以在读 Theia 代码的时候，我的习惯是：

- 一旦看到涉及“main/left/right/bottom area 布局”的地方，就自动联想到背后一定有 Lumino 的 DockPanel 在运作。  
- 遇到拖拽 / 合并 / 分屏相关逻辑时，会先在 Lumino 这几个布局 Widget 里找“底层行为”，再看 Theia 是怎么在上层包一层自己的抽象的。

## 小结：把这几块积木认清楚，再看 IDE 布局就不那么迷幻了

从 Widget -> signaling/messaging -> 布局 Widget（DockPanel / SplitPanel / TabBar）这一串看下来，大概可以得到一个还算清晰的图景：

- **Widget**：长什么样、放什么内容。  
- **signaling / messaging**：内部怎么“说话”和“按节奏动起来”。  
- **DockPanel / SplitPanel / TabBar**：这些砖最后怎么拼成一个能分屏、能拖拽、能记住布局的桌面感 UI。

对我来说，把这几块单独拎出来写一篇，是为了以后再遇到奇怪的布局问题（比如：某个视图拖不对地方 / 重启之后布局错乱），脑子里能快速定位：  
**到底是 Theia 应用层的“视图注册/还原”出了问题，还是 Lumino 那边的布局行为需要再去翻一遍源码。**

