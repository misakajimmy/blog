---
title: "InversifyJS 基础演示(TS)"
weight: 11
---

<iframe src="https://codesandbox.io/embed/xenodochial-babbage-tp3us?autoresize=1&fontsize=14&hidenavigation=1&module=%2Fsrc%2Ftype.ts&theme=dark"
     style="width:100%; height:800px; border:0; border-radius: 4px; overflow:hidden;"
     title="xenodochial-babbage-tp3us"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## 代码解析

```Typescript
import * as inversify from 'inversify';
import 'reflect-metadata';
```

引入 `inversify` 以及 `reflect-metadata` 两个包。

```Typescript
const TYPES = {
    // 忍者
    Ninja: 'Ninja',
    // 武士刀
    Katana: 'Katana',
    // 手里剑
    Shuriken: 'Shuriken'
}
```

定义了类对应的映射，这里定义了三个类，看过火影的应该很容易理解Ninja(忍者)、Katana(武士刀)和Shuriken(手里剑)之间的关系。

```Typescript
class Katana {
    hit() {
        return 'cut!';
    }
}

class Shuriken {
    throw() {
        return 'hit!';
    }
}

class Ninja {
    private _katana: Katana;
    private _shuriken: Shuriken;

    constructor(katana: Katana, shuriken: Shuriken) {
        this._katana = katana;
        this._shuriken = shuriken;
    }

    fight() {
        return this._katana.hit();
    }

    sneak() {
        return this._shuriken.throw();
    }
}
```

接着是Ninja(忍者)、Katana(武士刀)和Shuriken(手里剑)三个类的实现。其中 `Katana` 类实现了 `hit` 方法，`Shuriken` 实现了 `throw` 方法，`Ninja` 中定义了两个私有对象 `_katana` 和 `_shuriken`，并在构造函数之中传入这个两个对象，`Ninja` 类中的 `fight` 方法中将调用 `_katana` 的 `hit` 方法，`sneak` 方法中将调用 `_shuriken` 的 `throw` 方法。

```Typescript
// 声明可被注入以及其依赖项
inversify.decorate(inversify.injectable(), Katana);
inversify.decorate(inversify.injectable(), Shuriken);
inversify.decorate(inversify.injectable(), Ninja);
inversify.decorate(inversify.inject(TYPES.Katana), Ninja, 0);
inversify.decorate(inversify.inject(TYPES.Shuriken), Ninja, 1);

// 声明绑定关系
const container = new inversify.Container();
container.bind(TYPES.Ninja).to(Ninja);
container.bind(TYPES.Katana).to(Katana);
container.bind(TYPES.Shuriken).to(Shuriken);
```

//@TODO

这里终于引入了依赖注入的概念，这里使用了 `decorate` 来将 `Katana`、`Shuriken`和`Ninja`