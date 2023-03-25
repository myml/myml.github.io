---
title: 使用 Angular 构建可跨框架使用的 Web Components
date: 2022-06-24T02:21:00+08:00
draft: false
tags: ["angular"]
categories: ["前端"]
---

Angular的组件很棒，但只能在angular框架中使用，本文旨在记录使用 Angular 构建 Web Components 的最简步骤，[Web Components](https://developer.mozilla.org/zh-CN/docs/Web/Web_Components)是标准化的 HTML 组件规范，可实现跨框架(angular，vue，react)重用组件，甚至在无框架场景使用。

## 一：创建 angular 应用

本文假设你已经熟悉了 angular 框架的相关知识，并已安装 angular 开发的基本环境。

使用 `ng new --style scss --routing false m` 创建一个 angular 应用，这会等待一端时间，请耐心点。


<!--more-->

使用 `ng add @angular/elements` 安装 Angular Web Components 依赖库。

## 二：创建组件

本文尝试将angular material组件库中的日期选择器(datepicker)构建为Web Components，以实现在非angular环境使用。

使用 `ng add @angular/material` 安装 angular material 组件库，在 `app.modules.ts` 导入 `MatNativeDateModule` 和 `MatDatepickerModule` 。

```typescript
import { MatNativeDateModule } from '@angular/material/core';
import { MatDatepickerModule } from '@angular/material/datepicker';
```

添加 `MatNativeDateModule` 和 `MatDatepickerModule` 到 `AppModule` 模块的imports;

```typescript
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    MatDatepickerModule,
    MatNativeDateModule,
  ],
```

删除 `app.component.html` 文件的所有内容，写入

```html
<input [matDatepicker]="picker" />
<mat-datepicker-toggle matSuffix [for]="picker"></mat-datepicker-toggle>
<mat-datepicker #picker></mat-datepicker>
```

运行 `yarn start --open` 你应该能在弹出的浏览器上看到一个日历符号，点击日历符号后会弹出一个日期选择框，到此为止我们创建了一个基本的日期选择器的组件。

## 三：封装组件

上节我们创建了一个日期选择期，这节主要讲解如何将组件封装为Web Components。
从 `AppModule` 删除 `bootstrap` 组件（AppComponent），并添加构造函数和ngDoBootstrap钩子。

```typescript
// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    MatDatepickerModule,
    MatNativeDateModule,
  ],
  providers: [],
  bootstrap: [],
})
export class AppModule implements DoBootstrap {
  constructor(injector: Injector) {
    const dateComponent = createCustomElement(AppComponent, { injector });
    customElements.define('m-date', dateComponent);
  }
  ngDoBootstrap(appRef: ApplicationRef): void {}
}
```

构造函数中使用了 `@angular/elements` 的createCustomElement方法创建Web Components组件，然后使用Web Component规范中的 `customElements` 注册元素。

## 四：使用Web Components组件

由于在上节将 `AppComponent` 从 `bootstrap` 删除，`AppComponent` 不会再自动挂载到 `app-root`，可以在 `index.html` 删除 `app-root` 元素，添加 `m-date` 元素，以使用上节注册的组件。

```html
<!-- index.html -->
  <body>
    <m-date></m-date>
  </body>
```

由于angular默认并不会在修改 `index.html` 后 自动刷新页面，需要在浏览器按F5手动刷新页面，在刷新后应该可以看到日历图标，点击后会弹出日期选择对话框。在开发者调试工具中可以看到，日志图表位于 `m-date` 的元素中。

## 五：发布Web Components组件

上节中已经在angular的 `index.html` 粗略测试了封装的 Web Componetns的可用性，本节讲述如何在Linux 对封装的Web Components编译和打包。
使用 `yarn build` 编译后会产出以下几个文件。

```sh

Initial Chunk Files           | Names         |  Raw Size | Estimated Transfer Size
main.467a0ab37287e536.js      | main          | 365.53 kB |                86.67 kB
styles.af496c625d1d99e8.css   | styles        |  73.07 kB |                 7.55 kB
polyfills.a1cc02388ba313f2.js | polyfills     |  33.06 kB |                10.65 kB
runtime.68788f74f641e937.js   | runtime       |   1.03 kB |               595 bytes

                              | Initial Total | 472.68 kB |               105.45 kB
```

可以看到有3个js文件和一个css文件，如果直接将这四个文件发布出去，使用者需要分别导入这四个文件，显的十分麻烦。在 Linux 可以通过简单的命令实现文件的合并：

```bash
cat dist/m/*.js > main.js
cat dist/m/*.css > main.css
```

文件合并后只需要在html中引入一个js和一个css文件，即可使用 `m-date` 元素创建一个日期选择期。

```html
<html>
  <link rel="stylesheet" href="main.css" />
  <script src="main.js"></script>
  <body>
    <m-date></m-date>
  </body>
</html>
```

这两个文件可在任意现代化浏览器使用（永别了！IE），而无需在不同框架重复实现组件。

## 总结

本文通过封装一个日期选择期的例子，介绍了在angular中创建通用的Web Components组件的过程。介于篇幅有限，在后续文章中，再向大家介绍怎么在Web Components组件中使用 `Input` `Output` `slot`等高级功能。
