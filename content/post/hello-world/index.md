---
title: 第一篇文章
description: 一篇示例文章，演示 Stack 主题支持的常用 front matter 与 Markdown 特性
date: 2026-08-12
slug: hello-world
categories:
    - 示例分类
tags:
    - Hugo
    - Stack
toc: true
comments: false
draft: false
---

这是站点的第一篇文章。它是一个 **page bundle**——目录 `post/hello-world/` 下的
`index.md`，配图可以直接放同级目录，用相对文件名引用。

## Front matter 说明

主题会读取这些字段：

| 字段 | 作用 |
| --- | --- |
| `image` | 文章头图，同时用于 OpenGraph 和搜索索引 |
| `categories` / `tags` | 分类与标签，驱动侧边栏 widget 和相关文章 |
| `toc` | 是否显示目录，也可在 `hugo.toml` 全局开关 |
| `math` | 启用 KaTeX 公式渲染 |
| `slug` | 配合 `[permalinks]` 决定最终 URL |

## 代码高亮

`hugo.toml` 里设了 `noClasses = false`，因此语法高亮跟随亮/暗色主题切换：

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Hugo!")
}
```

## 写新文章

```sh
cd dev
./hugo.exe new content post/my-post/index.md
```

模板来自 `archetypes/default.md`，新建的文章默认 `draft = true`，
需要 `./hugo.exe server -D` 才能看到。
