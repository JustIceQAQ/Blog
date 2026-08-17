# JustIceQAQ Blog

[![Build and deploy](https://github.com/JustIceQAQ/Blog/actions/workflows/hugo.yaml/badge.svg)](https://github.com/JustIceQAQ/Blog/actions/workflows/hugo.yaml)
---

## 框架

- `Hugo` + `Blowfish`

## 新文章

- `hugo new posts/你的文章檔名.md`

## 服務啟動

- `hugo server --disableFastRender`

### 版本更新

- `Hugo`
    - `hugo mod get -u`

- `Blowfish`
    - `git submodule update --remote --merge`

### 自定義

- `c_use_expired_alert`
    - 下列地方設定 (優先度依次排列，越前面最優先)
        1. *.md 內設定
        2. params.toml > [article]

    - 可以決定是否要啟動"文章過舊"提醒   