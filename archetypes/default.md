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
    

```bash
$ hugo
```
