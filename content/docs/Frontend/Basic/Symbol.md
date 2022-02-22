---
title: "Symbol"
weight: 21
---

# Symbol

{{< tip >}}
原文链接：[developer.mozilla.org](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol)
{{< /tip >}}

**symbol** 是一种基本数据类型 （[primitive data type](https://developer.mozilla.org/zh-CN/docs/Glossary/Primitive)）。`Symbol()`函数会返回**symbol**类型的值，该类型具有静态属性和静态方法。它的静态属性会暴露几个内建的成员对象；它的静态方法会暴露全局的symbol注册，且类似于内建对象类，但作为构造函数来说它并不完整，因为它不支持语法："`new Symbol()`"。

<iframe src="https://interactive-examples.mdn.mozilla.net/pages/js/symbol-constructor.html"      
style="width:100%; height:700px; border:0; border-radius: 4px; overflow:hidden;"> </iframe>


## 语法

```Javascript
Symbol([description])
```

### 参数

- `description` `可选`

可选的，字符串类型。对symbol的描述，可用于调试但不是访问symbol本身。

## 描述

直接使用`Symbol()`创建新的symbol类型，并用一个可选的字符串作为其描述。

```Javascript
var sym1 = Symbol();
var sym2 = Symbol('foo');
var sym3 = Symbol('foo');
```

上面的代码创建了三个新的symbol类型。 注意，`Symbol("foo")` 不会强制将字符串 “foo” 转换成symbol类型。它每次都会创建一个新的 symbol类型：

```Javascript
Symbol("foo") === Symbol("foo"); // false
```

下面带有 [new](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new) 运算符的语法将抛出 [TypeError](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypeError) 错误：

```Javascript
var sym = new Symbol(); // TypeError
```

这会阻止创建一个显式的 Symbol 包装器对象而不是一个 Symbol 值。围绕原始数据类型创建一个显式包装器对象从 ECMAScript 6 开始不再被支持。 然而，现有的原始包装器对象，如 `new Boolean`、`new String`以及`new Number`，因为遗留原因仍可被创建。

如果你真的想创建一个 <u>Symbol 包装器对象</u> (`Symbol wrapper object`)，你可以使用 `Object()` 函数：

```Javascript
var sym = Symbol("foo");
typeof sym;     // "symbol"
var symObj = Object(sym);
typeof symObj;  // "object"
```

### 全局共享的 Symbol

上面使用`Symbol()` 函数的语法，不会在你的整个代码库中创建一个可用的全局的symbol类型。 要创建跨文件可用的symbol，甚至跨域（每个都有它自己的全局作用域） , 使用 [`Symbol.for()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for) 方法和  [`Symbol.keyFor()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/keyFor) 方法从全局的symbol注册表设置和取得symbol。

### 在对象中查找 Symbol 属性

[`Object.getOwnPropertySymbols()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols) 方法让你在查找一个给定对象的符号属性时返回一个symbol类型的数组。注意，每个初始化的对象都是没有自己的symbol属性的，因此这个数组可能为空，除非你已经在对象上设置了symbol属性。

## 属性

<!-- TODO -->