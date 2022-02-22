---
title: "InversifyJS"
weight: 3
---

一个强大又轻量的控制反转容器，提供给JavaScript 和 Node.js 应用使用，使用TypeScript编写。

## 简介

nversifyJS 是一个轻量的 (4KB) 控制反转容器 (IoC)，可用于编写 TypeScript 和 JavaScript 应用。 它使用类构造函数去定义和注入它的依赖。InversifyJS API 很友好易懂, 鼓励对 OOP 和 IoC 最佳实践的应用.

## 为什么要有 InversifyJS?

JavaScript 现在支持面向对象编程，基于类的继承。 这些特性不错但事实上它们也是 [危险的](https://medium.com/@dan_abramov/how-to-use-classes-and-sleep-at-night-9af8de78ccb4)。 我们需要一个优秀的面向对象设计（比如 [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design))，[Composite Reuse](https://en.wikipedia.org/wiki/Composition_over_inheritance)等）来保护我们避免这些威胁。然而，面向对象的设计是复杂的，所以我们创建了 InversifyJS。

InversifyJS 是一个工具，它能帮助 JavaScript 开发者，写出出色的面向对象设计的代码。

## 目标

InversifyJS有4个主要目标:

1. 允许JavaScript开发人员编写遵循 SOLID 原则的代码。

2. 促进并鼓励遵守最佳的面向对象编程和依赖注入实践。

3. 尽可能少的运行时开销。

4. 提供[艺术编程体验和生态](https://github.com/inversify/InversifyJS/blob/master/wiki/ecosystem.md)。

## 为什么要写这篇笔记

最近的项目中大量的运用到了 `InversifyJS` 所以有一些学习感悟心得想记录一下，本篇笔记大体上是基于 https://doc.inversify.cloud/zh_cn/ 这份中文文档来写的，比起原文档 `API` 字典式的记录 `InversifyJS` 的使用方法，本篇笔记更多的是主观的的记录自己的学习心得，以供自己以后回顾以及大家共同学习讨论。