---
title: "弹性盒子 align-items 与 align-content 的区别"
weight: 22
---

![flex-box-align](/images/flex-box-align.png)

最近在写的一个项目大量的用到了 `flexbox` ，但在子节点布局时，在使用 `align-content` 想要元素水平居中时经常不生效，在排查问题时确定父节点已经确定 `height` 和 `width` 则可以排除 `flexbox` 没有宽高的问题。翻阅文档发现 除了 `align-content` 外还有个 `align-items` 的css属性，这篇笔记则记录这两个属性的差别。

{{< tip >}}
本文参考了知乎大佬 Tabris 的文章，本文仅作为个人的学习记录

原文链接：[[前端]弹性盒子 align-items 与 align-content 的区别](https://zhuanlan.zhihu.com/p/87146411)
{{< /tip >}}

## 定义

- align-items：

    作用对象：弹性盒子容器(flex containers)；

    描述：该属性可以控制弹性容器中成员在当前行内的对齐方式。当成员设置了align-self 属性时，父容器的 align-items 值则不再对它生效；

- align-content：

    作用对象：多行弹性盒子容器(multi-line flex containers)；

    描述：当弹性容器在正交轴方向还存在空白时，该属性可以控制其中所有行的对齐方式。Note：该属性无法作用于单行 (flex-wrap: no-wrap) 弹性盒子；

## 对比

### 相同点

- 都被用来设置对齐行为。

### 不同点

- align-items 的设置对象是行内成员;
- align-content 的设置对象是所有行，且只有在多行弹性盒子容器中才生效。

## 举例

以 center 关键字为例。

文档对两个属性 cetner 关键字的描述：

- align-items：行内成员会在其边界盒正交轴上被居中（如果行正交尺寸小于行内成员尺寸，行内成员将会在正交轴两方向等量溢出）。[链接](https://www.w3.org/TR/2018/CR-css-flexbox-1-20181119/#valdef-align-items-centerr)；

- align-content：所有行被集中(挤)在弹性容器（正交轴）中间。它们彼此之间齐平，并且跟弹性盒子正交起始边界的空白与跟弹性盒子正交结束边界的空白相等。（如果溢出空白为负数，所有行将会在正交轴两方向等量溢出）[链接](https://www.w3.org/TR/2018/CR-css-flexbox-1-20181119/#valdef-align-content-center)。

列出所有会产生影响的条件，包括：是否自动换行 (flex-wrap: wrap/no-wrap)，一行和多行，留和不留多余空白。3个条件排列组合后有8中可能。