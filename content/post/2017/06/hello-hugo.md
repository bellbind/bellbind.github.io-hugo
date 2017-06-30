---
title: "Hello Hugo"
date: 2017-06-30T23:13:45+09:00
categories: ["2017-06"]
tags: ["hugo"]
draft: false
---

github pagesをhugoにした話。

<!--more-->

## hugo のインストール

macOSでhomebrewより:

```bash
$ brew install hugo
```

## hugoサイト作成

ソース側もgitで管理する:

```bash
$ hugo new site bellbind.github.io-hugo
$ cd bellbind.github.io-hugo
$ git init
$ git add .
$ git commit -m "[init]"
```

## テーマインストール

icarusを使いました:

```bash
$ git submodule add https://github.com/digitalcraftsman/hugo-icarus-theme.git themes/icarus
$ cp themes/icarus/exampleSite/config.toml .
$ cp themes/icarus/exampleSite/data/l10n.toml data/
```

まず、config.tomlの`themeDir`は削り、`theme = "icarus"`にすることで、

```bash
$ hugo server
```

で http://localhost:1313/ にてきちんと表示ができるようになる。

あとは、ブラウザでページを見ながら、
自分の情報に合わせてconfig.tomlを編集していく。

## 公開ページ

まず、すでにあるgithub pagesのリポジトリを、cloneしてくる:

```bash
$ git clone git@github.com:bellbind/bellbind.github.io.git public/
```

そのうえでhugoコマンドで生成し、commitしてpush:

```bash
$ hugo
$ cd public/
$ git add .
$ git commit -m "$(LANG=C date)"
$ git push
```


## 運用

記事ページは"content/post/2017/06/hello-hugo.md"におくことにした。
また、categoriesとして"2017-06"を入れることにした。

そのためarchitypes/default.mdを以下のように書いた。

```md
---
title: "{{ replace .TranslationBaseName "-" " " | title }}"
slug: "{{ .TranslationBaseName }}"
date: {{ .Date }}
categories: ["{{dateFormat "2006-01" .Date}}"]
tags: []
draft: false
---

TL;DR

<!--more-->

## sub title

- item 1
    - sub 1
    
```

