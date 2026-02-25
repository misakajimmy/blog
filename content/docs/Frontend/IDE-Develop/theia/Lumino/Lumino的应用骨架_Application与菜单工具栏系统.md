---
title: "Lumino 的应用骨架：Application + 菜单 / 工具栏 / 右键菜单"
weight: 6
---

> 前面几篇更多是把一块块“砖”（Widget、布局、command 等）单独拎出来看，  
> 这篇换个视角：**把这些东西装进一个完整的 Lumino 应用里**，顺带看看菜单栏、工具栏、右键菜单这些常见 UI 是怎么围绕 command 系统一起工作的。

## `@lumino/application`：把零散的能力装进一个 App

如果只用 `@lumino/widgets`，你完全可以手动 new 一个 `DockPanel` / `BoxPanel`，往里塞 Widget，然后挂到 `document.body` 上。  
但一旦你想要：

- 全局的 `CommandRegistry` 和快捷键处理  
- 顶部菜单栏（`MenuBar`）  
- 右键菜单（`ContextMenu`）  
- 工具栏、命令面板之类的“全局 UI”

就会发现需要一个更高级的“应用骨架”来帮你把这些拼在一起，这就是 `@lumino/application` 要做的事情。

它的核心概念通常包括：

- **`Application`**：整个 Lumino 应用的入口，负责启动、挂载、管理命令等。  
- **`Shell`**：应用的外壳，内部才是我们熟悉的布局（通常会内嵌一个 DockPanel 等）。  

在 Theia 里你也能看到类似的影子：  
有一个 Application / FrontendApplication 的东西启动整套系统，然后有 ApplicationShell 作为 UI 壳。

## 一个极简 Lumino Application 骨架

很多示例会这么写一个最小壳（伪代码略简化，只看结构）：

```ts
import { CommandRegistry } from '@lumino/commands';
import { DockPanel, Widget } from '@lumino/widgets';
import { Application } from '@lumino/application';

// 1. 定义一个简单的 Shell（内部有一个 DockPanel）
class SimpleShell extends Widget {
  readonly dock: DockPanel;

  constructor() {
    super();
    this.addClass('my-SimpleShell');

    this.dock = new DockPanel();
    this.dock.id = 'main-dock';

    // 把 dock 的 DOM 挂到 shell 的 node 上
    this.node.appendChild(this.dock.node);
  }
}

// 2. 定义 Application，指定 Shell 和 CommandRegistry
class SimpleApplication extends Application<SimpleShell> {
  constructor(options: Application.IOptions<SimpleShell>) {
    super(options);
  }
}

// 3. 启动应用
const commands = new CommandRegistry();
const shell = new SimpleShell();

const app = new SimpleApplication({
  shell,
  commands,
});

window.addEventListener('load', () => {
  app.start(); // 挂载并启动整个应用
});
```

这里很多细节可以展开讲，但对我们来说，先记住两点就够了：

- **Application 负责 glue code**：把命令、快捷键、菜单、shell 这些能力粘在一起。  
- **Shell 负责布局容器**：内部才是 DockPanel / SplitPanel / TabBar 等。

接下来就可以往这个骨架上挂菜单栏、工具栏、右键菜单了。

## 菜单栏：`Menu` + `MenuBar`

菜单栏是 Lumino 里和 command 系统结合得最紧的一块 UI。  
基本思路是：

- 用 `Menu` 代表一个下拉菜单（比如「文件」「编辑」）。  
- 把这些 `Menu` 挂到 `MenuBar` 上。  
- 每个菜单项对应一个命令 id。

一个简化示例：

```ts
import { CommandRegistry } from '@lumino/commands';
import { Menu, MenuBar } from '@lumino/widgets';

const commands = new CommandRegistry();

commands.addCommand('file:new', {
  label: 'New File',
  execute: () => {
    console.log('New file');
  },
});

commands.addCommand('file:open', {
  label: 'Open File',
  execute: () => {
    console.log('Open file');
  },
});

// 创建一个“文件”菜单
const fileMenu = new Menu({ commands });
fileMenu.title.label = 'File';
fileMenu.addItem({ command: 'file:new' });
fileMenu.addItem({ command: 'file:open' });

// 创建菜单栏并挂上去
const menuBar = new MenuBar();
menuBar.addMenu(fileMenu);

document.body.appendChild(menuBar.node);
```

要点：

- `Menu` 和 `MenuBar` 都依赖一个 `CommandRegistry` 实例，通过 command id 来渲染菜单项的 label / enabled 状态等。  
- 一旦命令的 `label`/`isEnabled` 逻辑变了，菜单 UI 会自动跟着更新。

在 Theia 里，菜单栏同样是围绕 command 系统构建的，只不过多了层 `MenuContribution` / 配置化的菜单树；但底层这条“命令驱动菜单项”的思路是一致的。

## 右键菜单：`ContextMenu`

右键菜单的使用体验和菜单栏类似，但触发方式是基于鼠标事件 + DOM 选择器。  
Lumino 提供了一个 `ContextMenu` 帮你管理：

```ts
import { CommandRegistry } from '@lumino/commands';
import { ContextMenu } from '@lumino/widgets';

const commands = new CommandRegistry();

commands.addCommand('editor:copy', {
  label: 'Copy',
  execute: () => {
    console.log('Copy!');
  },
});

const contextMenu = new ContextMenu({ commands });

// 针对特定区域注册右键菜单项
contextMenu.addItem({
  command: 'editor:copy',
  selector: '.my-Editor',
  rank: 1,
});

// 全局监听 contextmenu 事件
window.addEventListener('contextmenu', event => {
  contextMenu.open(event);
});
```

这里的关键点有两个：

- `selector` 决定了“在哪些 DOM 元素上右键，才会显示这条菜单项”。  
- `ContextMenu` 内部同样是通过 command id 去取 label / isEnabled / execute。

从 Theia 的视角看，右键菜单也是类似的套路：  
不同 view/contribution 注册自己的上下文菜单项，最终映射到同一个命令系统上。

## 工具栏：一组“命令按钮”的容器

Lumino 没有强制你必须用某种 Toolbar 类型，你可以：

- 直接用某个 Panel / Widget 当作工具栏容器；  
- 在里面放一堆“命令按钮”——比如前面那篇 command 文里示意的 `CommandButton` Widget。

一个非常简化的伪代码例子：

```ts
import { CommandRegistry } from '@lumino/commands';
import { Panel, Widget } from '@lumino/widgets';

class CommandButton extends Widget {
  constructor(
    private commands: CommandRegistry,
    private commandId: string
  ) {
    super({ node: document.createElement('button') });
    this.addClass('my-CommandButton');
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
      void this.commands.execute(this.commandId);
    }
  }

  private _render(): void {
    const label = this.commands.label(this.commandId) ?? this.commandId;
    (this.node as HTMLButtonElement).textContent = label;
    (this.node as HTMLButtonElement).disabled = !this.commands.isEnabled(this.commandId);
  }
}

// 工具栏容器
const toolbar = new Panel();
toolbar.id = 'main-toolbar';

toolbar.addWidget(new CommandButton(commands, 'file:new'));
toolbar.addWidget(new CommandButton(commands, 'file:open'));
```

本质上，**工具栏只是“水平摆了一排命令按钮的 Panel”**，  
真正的行为还是集中在 command 系统里。

在 Theia 中你也会看到各种 View/Editor 的 toolbar，本质上也是给某一组命令提供一块更显眼的入口区域。

## 把这些拼回 Application 视角

如果把前面的东西都装回 `Application` 里，一个简化的心智模型大概是这样：

- Application 创建并持有：
  - 一个 `CommandRegistry`（命令与快捷键）。  
  - 一个 Shell（里面是 DockPanel 等布局）。  
  - 一个 `MenuBar` + 一个 `ContextMenu` + 若干工具栏 Widget。
- Shell 负责给布局和各种视图（Widget）腾位置。  
- 菜单栏 / 右键菜单 / 工具栏，仅仅是围绕 `CommandRegistry` 渲染 UI 的不同外壳。

Theia 则在这个基础上又往上盖了一层：

- 有自己的 `CommandContribution` / `MenuContribution` 接口，让扩展可以注册命令和菜单，而不用直接操作 Lumino 的 API。  
- 有抽象出来的 ApplicationShell，把 Lumino Shell 的细节再包一层，让 IDE 级的需求（多语言支持、view/container 抽象等）更好处理。

## 小结：Application + 菜单体系 = 把“命令世界”塞进一个真正的 App

到这篇为止，跟 Lumino 相关的几条主干大概就比较齐了：

- Widget / 布局系统：东西长什么样，摆在哪儿。  
- signaling / messaging：状态和生命周期的流动。  
- command 系统：行为的抽象与复用。  
- Application + 菜单 / 工具栏 / 右键菜单：把这些能力装进一个“完整应用壳”里。

对我来说，理解 `@lumino/application` 这一层最大的好处是：  
**再看 Theia 的启动流程和 Shell 代码时，不会把“IDE 自己的那一层”与“Lumino 提供的通用骨架”混在一起，看见 Application/Shell/Menu/Command 这些词汇，脑子里都有一个明确的 Lumino 版本可以对照。**

