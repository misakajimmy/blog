---
title: "Lerna"
weight: 12
---

{{< tip >}}
项目链接：[github.com/lerna/lerna](https://github.com/lerna/lerna)
{{< /tip >}}

[Lerna](https://github.com/lerna/lerna) 是一个管理工具，用于管理包含多个软件包（package）的 JavaScript 项目。

## 关于

将大型代码仓库分割成多个独立版本化的 软件包（package）对于代码共享来说非常有用。但是，如果某些更改 跨越了多个代码仓库的话将变得很 麻烦 并且难以跟踪，并且， 跨越多个代码仓库的测试将迅速变得非常复杂。

为了解决这些（以及许多其它）问题，某些项目会将 代码仓库分割成多个软件包（package），并将每个软件包存放到独立的代码仓库中。但是，例如 [Babel](https://github.com/babel/babel/tree/master/packages), [React](https://github.com/facebook/react/tree/master/packages), [Angular](https://github.com/angular/angular/tree/master/modules),
[Ember](https://github.com/emberjs/ember.js/tree/master/packages), [Meteor](https://github.com/meteor/meteor/tree/devel/packages), [Jest](https://github.com/facebook/jest/tree/master/packages), 等项目以及许多其他项目则是在 一个代码仓库中包含了多个软件包（package）并进行开发。

**Lerna is a tool that optimizes the workflow around managing multi-package
repositories with git and npm.**

Lerna 还可以减少开发和构建环境中大量包副本的时间和空间需求——这通常是将项目分成许多单独的 NPM 包的缺点。有关详细信息，请参阅 [hoist documentation](doc/hoist.md)。

### Lerna repo 是什么样的？

它实际上很简单。您的文件结构如下所示：

```
my-lerna-repo/
  package.json
  packages/
    package-1/
      package.json
    package-2/
      package.json
```

### Lerna 能做什么？

Lerna 中的两个主要命令是 `lerna bootstrap` 和 `lerna publish`。

`bootstrap` 将 repo 中的依赖项链接在一起。 `publish` 将帮助发布任何更新的包。

### Lerna 不能做什么？

Lerna 不是 serverless monorepos 的部署工具。Hoisting 可能与传统的无服务器 monorepo 部署技术不兼容。

## 入门

{{< tip >}}
本着用新不用旧的原则，本文像官方文档学习，使用 Lerna 3.0。

比起npm作者偏爱用yarn，如果不喜欢yarn的可以参考 [`yarn` 与 `npm` 命令对比](../yarn/#yarn-与-npm-命令对比)
{{< /tip >}}

推荐全局安装 Lerna

```sh
yarn add -g lerna
```

接下来，我们将创建一个新的 git 代码仓库：

```sh
git init lerna-repo && cd lerna-repo
```

现在，我们将上述仓库转变为一个 Lerna 仓库：

```sh
lerna init
```

这将创建一个`lerna.json`配置文件和一个`packages`文件夹，因此您的文件夹现在应如下所示：

```
lerna-repo/
  packages/
  package.json
  lerna.json
```

## 如何使用Lerna

Lerna 允许您使用以下两种模式之一来管理您的项目：`Fixed` 或 `Independent`。

### Fixed/Locked 模式 (default)

`Fixed`模式 Lerna 项目在单个版本行上运行。版本保存在`lerna.json`项目根目录下的文件中`version`。当你运行时`lerna publish`，如果一个模块自上次发布以来已经更新，它将更新到你发布的新版本。这意味着您只在需要时发布新版本的包。

{{< tip >}}
注意：如果您的主要版本为零，则所有更新都被认为是[破坏性](https://semver.org/#spec-item-4)的。因此，`lerna publish`使用主要版本零运行并选择任何非预发布版本号将导致为所有包发布新版本，即使自上次发布以来并非所有包都已更改。
{{< /tip >}}

这是Babel当前使用的模式。如果您想自动将所有包版本绑定在一起，请使用此选项。这种方法的一个问题是，任何包的重大变化都会导致所有包都有一个新的主要版本。

### Independent 模式

`lerna init --independent`

独立模式 Lerna 项目允许维护人员彼此独立地增加包版本。每次发布时，您都会收到有关已更改的每个包的提示，以指定它是补丁、次要、主要还是自定义更改。

独立模式允许您更具体地更新每个包的版本，并且对一组组件有意义。将此模式与语义释放之类的东西结合起来会减轻痛苦。（[atlassian/lerna-semantic-release](https://github.com/atlassian/lerna-semantic-release)已经有这方面的工作）。

{{< tip >}}
将`lerna.json`中的`version`设置为`independent`以运行Independent模式。
{{< /tip >}}

## 基本概念

Lerna在运行命令时遇到错误会记录到一个`lerna-debug.log`文件（与 `npm-debug.log` 相同）。

Lerna 还支持[scoped packages](https://docs.npmjs.com/misc/scope)。

### lerna.json

```json
{
  "version": "1.1.3",
  "npmClient": "yarn",
  "command": {
    "publish": {
      "ignoreChanges": ["ignored-file", "*.md"],
      "message": "chore(release): publish",
      "registry": "https://npm.pkg.github.com"
    },
    "bootstrap": {
      "ignore": "component-*",
      "npmClientArgs": ["--no-package-lock"]
    }
  },
  "packages": ["packages/*"]
}
```

- `version`: 存储库的当前版本
- `npmClient`: 用于指定特定客户端以运行命令的选项（也可以在每个命令的基础上指定）。更改为`"yarn"`使用 yarn 运行所有命令。默认为`“npm”`
- `command.publish.ignoreChanges`: 不包含在lerna changed/publish. 使用它来防止不必要地发布新版本以进行更改，例如修复`README.md`错字
- `command.publish.message`: 执行版本更新以进行发布时的自定义提交消息。有关更多详细信息，请参阅[@lerna/version](commands/version#--message-msg)
- `command.publish.registry`: 使用它来设置要发布到的自定义注册表 url 而不是 npmjs.org，如果需要，您必须已经过身份验证
- `command.bootstrap.ignore`: `lerna bootstrap` 运行命令时不会引导的 glob 数组
- `command.bootstrap.npmClientArgs`: 在 `lerna bootstrap` 运行时将传递到 `npm install` 的数据
- `command.bootstrap.scope`: 一个用于 `lerna bootstrap` 运行时，限制在运行命令时将引导哪些包的一个 glob 数组
- `packages`: 用作包位置的 glob 数组

配置文件`lerna.json`中的`packages`字段一个 glob 数组用来匹配拥有`package.json`的文件夹，这是 lerna 识别"子"包的方式（相对于项目"根"目录 `package.json`，它旨在管理整个 repo 的开发依赖项和脚本）。

默认情况下，lerna 将包列表初始化为`["packages/*"]`，但您也可以使用其他目录，例如`["modules/*"]`, 或`["package1", "package2"]`. 定义的 glob 与所在的目录相关，`lerna.json`通常是存储库根目录。唯一的限制是您不能直接嵌套包位置，但这也是“普通”npm 包共享的限制。

例如，`["packages/*", "src/**"]`匹配这棵树：

```
packages/
├── foo-pkg
│   └── package.json
├── bar-pkg
│   └── package.json
├── baz-pkg
│   └── package.json
└── qux-pkg
    └── package.json
src/
├── admin
│   ├── my-app
│   │   └── package.json
│   ├── stuff
│   │   └── package.json
│   └── things
│       └── package.json
├── profile
│   └── more-things
│       └── package.json
├── property
│   ├── more-stuff
│   │   └── package.json
│   └── other-things
│       └── package.json
└── upload
    └── other-stuff
        └── package.json
```

在下面定位叶子包`packages/*`被认为是“最佳实践”，但不是使用 Lerna 的要求。

#### 废弃字段

有些`lerna.json`字段不再使用。值得注意的包括：

- `lerna`: 最初用于表示 Lerna 的当前版本。在 v3 中 [过时](https://github.com/lerna/lerna/pull/1122) 并且 [删除](https://github.com/lerna/lerna/pull/1225)

### 常见的`devDependencies`

大多数`devDependencies`可以拉到 Lerna repo 的根目录`lerna link convert`

上述命令将自动 hoist things 并使用相关 `file:` 说明符。

Hoisting 有几个好处：

- 所有包都使用相同版本的给定依赖项
- 可以使用Snyk等自动化工具使根目录中的依赖项保持最新
- 减少依赖安装时间
- 需要更少的存储空间

请注意，`devDependencies`提供 npm 脚本使用的“二进制”可执行文件仍然需要直接安装在使用它们的每个包中。

例如，在这种情况下，为了 `lerna run nsp`（以及在包的目录中 `npm run nsp`）正常工作，`nsp`依赖项是必需的：

```json
{
  "scripts": {
    "nsp": "nsp"
  },
  "devDependencies": {
    "nsp": "^2.3.3"
  }
}
```

### Git 托管的依赖项

Lerna 允许将本地依赖包的目标版本编写为带有（例如`#committish#v1.0.0`或`#semver:^1.0.0` ）的git 远程 url ，而不是正常的数字版本范围。当包必须是私有的并且不需要私有 npm 注册表时，这允许通过 git 存储库分发包。

请注意，lerna 不会将git 历史记录实际拆分为单独的只读存储库。这是用户的责任。

```
// packages/pkg-1/package.json
{
  name: "pkg-1",
  version: "1.0.0",
  dependencies: {
    "pkg-2": "github:example-user/pkg-2#v1.0.0"
  }
}

// packages/pkg-2/package.json
{
  name: "pkg-2",
  version: "1.0.0"
}
```

在上面的例子中，

- `lerna bootstrap`将正确符号链接`pkg-2`到`pkg-1`.
- `lerna publish`将在`pkg-2`更改时更新`pkg-1` committish (`#v1.0.0`)

## 自述文件徽章

正在使用Lerna？添加一个 README 徽章来展示它：[![lerna](https://img.shields.io/badge/maintained%20with-lerna-cc00ff.svg)](https://lerna.js.org/)

```
[![lerna](https://img.shields.io/badge/maintained%20with-lerna-cc00ff.svg)](https://lerna.js.org/)
```