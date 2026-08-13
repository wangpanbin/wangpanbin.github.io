---
title: Hugo 静态站点用 Dart Sass 部署到 GitHub Pages
description: withdeploy 版 Hugo 没有 LibSass，踩过的 SCSS 编译坑和 CI 配 Dart Sass 的正确姿势
date: 2026-08-13
slug: hugo-dart-sass-deploy
categories:
    - 工具链
tags:
    - Hugo
    - GitHub Pages
    - SCSS
toc: true
draft: false
---

Hugo 0.153.0 弃用了 LibSass，0.153.2 之后 `extended` 二进制甚至不再带 LibSass——SCSS 编译成了"你需要自己装 transpiler"。

## 现象

```text
TOCCSS: failed to transform "/scss/style.scss" ... you need the extended version
```

或者（更精确的报错）：

```text
failed to transpile SCSS: sass not found
```

## 修法

两步，缺一不可。

### 1. 装 Dart Sass

本地：

```sh
npm install -g sass-embedded
```

**注意**：用 `sass-embedded`，不要用 `sass`。`sass` 包是 JS 包装脚本，Hugo exec 二进制时拿到的不是真正的 CLI、协议握手就报 `got unexpected EOF`。`sass-embedded` 自带 Dart 运行时和原生 snapshot。

CI（GitHub Actions）：

```yaml
- name: Install Dart Sass
  env:
    DART_SASS_VERSION: "1.83.0"
  run: |
    curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
    tar -C "$HOME/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
    echo "$HOME/.local/dart-sass" >> "$GITHUB_PATH"
```

Dart Sass 的版本要 ≥ 1.79，否则 `silenceDeprecations` 里某些 ID 会被忽略。

### 2. 覆盖主题的 Head style 模板

在 `dev/layouts/_partials/head/style.html` 里写：

```go-html-template
{{- $opts := dict
    "transpiler" "dartsass"
    "targetPath" "css/style.css"
    "silenceDeprecations" (slice "import" "global-builtin" "color-functions")
-}}
{{- $style := resources.Get "scss/style.scss" | toCSS $opts | minify | fingerprint "sha256" -}}
<link rel="stylesheet" href="{{ $style.RelPermalink }}">
```

不写这个 override，Hugo 默认走 libsass，extended 和 standard 都会失败。

## GitHub Actions 启用 Pages 还有个坑

第一次部署时 `actions/configure-pages@v5` 必须传 `enablement: true`，否则 deploy job 报 success 但 `pages/builds/latest` 永远 404。

```yaml
- name: Configure Pages
  uses: actions/configure-pages@v5
  with:
    enablement: true
```

## 验证

构建命令：

```sh
hugo --gc --minify
```

本地构建零 ERROR 零 WARN、产物里有 `public/css/style.min.<hash>.css` 大小正常（几 KB 到几十 KB），就证明整条链路通了。

CI 端用服务端 fetch 看站点是否能拉到样式、CSS 是否生效；如果拉到页面 HTML 但样式丢了，八成是 Dart Sass 没装对。
