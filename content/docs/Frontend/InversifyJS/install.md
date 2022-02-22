---
title: "项目中引入 InversifyJS"
weight: 10
---

## 安装

可以通过 `npm` 或者 `yarn` 来引入 `InversifyJS`：

```yarn
yarn add inversify reflect-metadata
```

{{< tip "warning" >}}
InversifyJS 类型定义包含在 inversify npm 包中。InversifyJS 需要 experimentalDecorators、 emitDecoratorMetadata 和 lib 编译选项开启在 tsconfig.json 文件中。
{{< /tip >}}

```json
{
    "compilerOptions": {
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    }
}
```

InversifyJS 需要一个支持下面特性的现代 JavaScript 引擎：

- [Reflect metadata](https://rbuckton.github.io/reflect-metadata/)
- [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) (仅在使用 [provider injection](https://github.com/inversify/InversifyJS#injecting-a-provider-asynchronous-factory) 时需要)
- [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) (仅在使用 [activation handlers](https://github.com/inversify/InversifyJS/blob/master/wiki/activation_handler.md) 时需要)

如果你的环境不支持其中的特性，那么需要引入 shim 或者 polyfill。

## 环境支持和 polyfills

InversifyJS 需要一个现代的 JavaScript 引擎，需要支持 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)、 [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)、 [Metadata Reflection API](https://rbuckton.github.io/reflect-metadata/) 和 [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 对象。 如果你的环境不支持这些的化，就需要引入 shim 或者 polyfill。

### Metadata Reflection API

{{< tip "warning" >}}
reflect-metadata polyfill 仅在进入应用时需要引入一次，因为反射对象是一个全局单例。更多详情参见[这里](https://github.com/inversify/InversifyJS/issues/262#issuecomment-227593844)。
{{< /tip >}}

始终需要。使用 [reflect-metadata](https://www.npmjs.com/package/reflect-metadata) 作为 polyfill。

```shell
yarn install reflect-metadata
```

`reflect-metadata` 的类型定义已在 `npm` 包中被包含。你需要添加如下引用到 `tsconfig.json` 中的类型字段：

```json
{
    "compilerOptions": {
        "types": ["reflect-metadata"]
    }
}
```

最后，引入 `reflect-metadata`。如果你使用：

```javascript
import "reflect-metadata";
```

如果你在使用网页浏览器那么可以使用 script 标签：

```HTML
<script src="./node_modules/reflect-metadata/Reflect.js"></script>
```

这将创建全局反射对象。

### Map

[Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) 在使用 InversifyJS 3 或者更高版本时需要。

大多数现代 JavaScript 引擎支持 map，但如果你需要支持旧浏览器，那么需要使用 map polyfill (比如 [es6-map](https://www.npmjs.com/package/es6-map
))。

### Promise

Promise 仅在使用 注入提供者 时需要。

多数现代 JavaScript 引擎支持 promise，但如果你需要支持就浏览器，那么需要使用 promise polyfill (比如 [es6-promise](https://www.npmjs.com/package/es6-map) 或者 [bluebird](https://www.npmjs.com/package/bluebird))。
