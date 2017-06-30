---
title: "Hugo Support"
date: 2017-07-01T02:47:01+09:00
categories: ["2017-07"]
tags: ["hugo", "github", "bash"]
draft: false
---

hugo用支援コマンドを作った。

<!--more-->

- https://github.com/bellbind/bellbind.github.io-hugo の/bin/以下

## bin/post

```bash
#!/bin/bash

#RUN=echo
TITLE=$1
DAY=$(date "+%d")
if [[ -z $TITLE ]]
then
    TITLE=$DAY
fi
POST=$(date "+post/%Y/%m")/${TITLE}.md
$RUN hugo new $POST
$RUN ${EDITOR=emacs} content/$POST
```

`./bin/post hugo-support` または `./bin/post` (日付で`01.md`になる)のように使う。

## bin/sync

```bash
#!/bin/bash

#RUN=echo

DATE=$(LANG=C date)
if $RUN hugo
then
   $RUN git add content
   $RUN git commit -m "Add post: $DATE"
   $RUN git push
   $RUN cd public/
   $RUN git add .
   $RUN git commit -m "Add post: $DATE"
   $RUN git push
fi
```

`./bin/sync` で使う。
