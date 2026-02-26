---
title: "在项目中引入 InversifyJS：配置与依赖"
weight: 3
---

> 这篇主要记录一下在 TypeScript 项目里引入 InversifyJS 需要做的配置，以及一些容易踩的坑。  
> 如果你在写 Theia 扩展，这些配置通常已经在 Theia 的模板项目里配好了，但理解一下背后的原理，对排查问题会很有帮助。

## 安装依赖

通过 `npm` 或 `yarn` 安装 InversifyJS 和它的必需依赖：

```bash
yarn add inversify reflect-metadata
# 或
npm install inversify reflect-metadata
```

两个包的作用：

- **`inversify`**：InversifyJS 的核心库，包含 `Container`、装饰器等。
- **`reflect-metadata`**：提供元数据反射 API 的 polyfill，InversifyJS 依赖它来读取装饰器生成的类型信息。

## TypeScript 配置：开启装饰器支持

InversifyJS 依赖 TypeScript 的装饰器和元数据反射功能，所以需要在 `tsconfig.json` 里开启相关选项：

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "types": ["reflect-metadata"]
  }
}
```

几个关键选项：

- **`experimentalDecorators: true`**：启用装饰器语法（`@injectable()`、`@inject()` 等）。
- **`emitDecoratorMetadata: true`**：让 TypeScript 在编译时生成类型元数据，这样 InversifyJS 才能知道构造函数参数的类型。
- **`types: ["reflect-metadata"]`**：引入 `reflect-metadata` 的类型定义。

{{< tip "warning" >}}
如果 `emitDecoratorMetadata` 没开启，InversifyJS 就无法自动推断构造函数参数的类型，必须手动用 `@inject(Symbol)` 标记每个参数，否则会报错。
{{< /tip >}}

## 引入 reflect-metadata

在应用入口文件的最顶部（在任何其他 import 之前），必须先引入 `reflect-metadata`：

```typescript
import "reflect-metadata"; // 必须在最前面！

import { Container, injectable, inject } from "inversify";
// ... 其他代码
```

{{< tip "warning" >}}
`reflect-metadata` 必须在所有其他代码之前引入，因为它会修改全局的 `Reflect` 对象。如果引入顺序不对，装饰器可能无法正常工作。
{{< /tip >}}

如果在浏览器环境里用 `<script>` 标签引入：

```html
<script src="./node_modules/reflect-metadata/Reflect.js"></script>
<script src="./your-app.js"></script>
```

## 环境支持与 polyfills

InversifyJS 需要一些现代 JavaScript 特性，大多数现代环境都支持，但如果你需要支持旧浏览器，可能需要 polyfill：

### 必需特性

- **Metadata Reflection API**：通过 `reflect-metadata` 包提供（已安装）。
- **Map**：InversifyJS 3+ 需要，现代浏览器都支持。如果需要支持旧浏览器，可以用 [es6-map](https://www.npmjs.com/package/es6-map) 作为 polyfill。

### 可选特性（仅在特定场景需要）

- **Promise**：仅在用到“provider injection”（异步工厂注入）时需要。现代环境都支持，旧浏览器可以用 [es6-promise](https://www.npmjs.com/package/es6-promise) 或 [bluebird](https://www.npmjs.com/package/bluebird)。
- **Proxy**：仅在用到“activation handlers”（激活处理器）时需要。现代环境都支持，旧浏览器可以用 [proxy-polyfill](https://www.npmjs.com/package/proxy-polyfill)。

## 在 Theia 项目中的情况

如果你在写 Theia 扩展，通常不需要自己配置这些：

- Theia 的模板项目（通过 `yo theia-extension` 生成）已经配好了 `tsconfig.json`。
- `reflect-metadata` 已经在 Theia 的入口文件里引入了。
- 你只需要在扩展代码里 `import { injectable, inject } from "inversify"` 就能直接用。

但理解这些配置的原理，对排查“为什么我的装饰器不生效”“为什么容器解析失败”这类问题会很有帮助。

## 常见问题排查

如果遇到装饰器不生效的问题，按这个顺序检查：

1. **`reflect-metadata` 是否在最前面引入？**  
   检查入口文件，确保 `import "reflect-metadata"` 在所有其他 import 之前。

2. **`tsconfig.json` 配置是否正确？**  
   确认 `experimentalDecorators` 和 `emitDecoratorMetadata` 都是 `true`。

3. **TypeScript 版本是否太旧？**  
   InversifyJS 需要 TypeScript 2.0+，建议用 3.0+ 以获得更好的装饰器支持。

4. **构建工具配置是否正确？**  
   如果用的是 webpack/vite 等，确保它们能正确处理装饰器语法（通常需要配合 `ts-loader` 或 `@babel/preset-typescript`）。
