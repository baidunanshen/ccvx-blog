---
description: 介绍一下自定义主题的部分样式
title: 🔧 主题样式定制
readingTime: false
tag:
 - 配置
recommend: 3
date: 2026-10-25
---

# 样式配置
样式自定义，参考[官方文档思路](https://vitepress.dev/guide/extending-default-theme#customizing-css)

在 `.vitepress/theme/index.ts` 中引入自定义的样式文件，覆盖默认主题样式即可

例如:

博客模板里，提供了一个如下例子

```ts
// .vitepress/theme/index.ts
import BlogTheme from '@sugarat/theme'

// 自定义样式重载 // [!code focus]
import './style.css' // [!code focus]

export default BlogTheme
```

里面有如下内容
```css
/* 定义遮罩样式 */
.VPHome {
  background: radial-gradient(ellipse, rgba(var(--bg-gradient-home), 1) 0%, rgba(var(--bg-gradient-home), 0) 150%);
}

/* 自定义首页背景图 */
.VPHome::before {
  background-image: url(./assets/bg.webp);
  background-size: cover;
}
```
解除文件`index.ts` 中`import './style.css'` 注释后，就能看到模板首页背景图发生了变化

![](https://img.cdn.sugarat.top/mdImg/MTY3Njk5MTAzODkzOQ==676991038939~fmt.webp)

## 首页背景
```css
/* 自定义首页背景图 */
.VPHome::before {
  background-image: url(./assets/bg.webp);
  background-size: cover;
}

.VPHome {
  /* 定义遮罩样式，控制图片展示的程度 */
  background: radial-gradient(
    ellipse,
    rgba(var(--bg-gradient-home), 1) 0%,
    rgba(var(--bg-gradient-home), 0) 150%
  );
}
```

## 置顶样式
可以自行修改置顶icon的样式
```css
/* 修改置顶icon的颜色 */
.blog-item .pin.pin::before {
  background-image: linear-gradient(red, red);
}
```
![](https://img.cdn.sugarat.top/mdImg/MTY3NzA3OTExMjgxMA==677079112810~fmt.webp)

```css
/* 隐藏置顶的icon */
.blog-item .pin.pin::before {
  display: none;
}
```
![](https://img.cdn.sugarat.top/mdImg/MTY3NzA3OTIwODAzNg==677079208036~fmt.webp)


## 主题色
vitepress 最新的默认主题色偏紫色，看着感觉有点不习惯

![](https://img.cdn.sugarat.top/mdImg/MTY5MTkyODQ0ODUzOQ==691928448539~fmt.webp)

主题包内置了一些主题色，可以自行选择，详见[全局配置:themeColor](https://theme.sugarat.top/config/global.html#themecolor)

如果内置的不满足要求，也可以进行自定义，自定义的方式是通过覆盖变量的方式

例如可以创建`user-theme.css`文件，内容如下
```css
/* 所有变量：https://github.com/vuejs/vitepress/blob/main/src/client/theme-default/styles/vars.css */
/* 自定义主题色示例，如下 */

/* 浅色模式使用的变量 */
html[theme] {
  --vp-c-user-1:red;
  --vp-c-user-2:blue;
  --vp-c-user-3:green;
  --vp-c-user-soft:rgba(255,0,0,.5);
}

/* 深色模式使用的变量 */
html[theme].dark {
  --vp-c-user-1:yellow;
  --vp-c-user-2:purple;
  --vp-c-user-3:orange;
  --vp-c-user-soft:rgba(255,255,0,.5);
}

/* 覆盖默认变量 */
html[theme],html[theme].dark {
  --vp-c-brand-1: var(--vp-c-user-1);
  --vp-c-brand-2: var(--vp-c-user-2);
  --vp-c-brand-3: var(--vp-c-user-3);
  --vp-c-brand-soft: var(--vp-c-user-soft);
}
```
将其引入到`.vitepress/theme/index.ts`中即可

```ts
import BlogTheme from '@sugarat/theme'

// 导入自定义主题色 // [!code focus]
import './user-theme.css' // [!code focus]

export default BlogTheme
```

## More
... wait a moment