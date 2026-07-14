---
title: "イメージダイジェストをDocker CLIもなく、イメージもダウンロードせずに調べる方法"
date: 2023-01-05T21:30:12+09:00
tags: ["carvel", "kbld"]
thumbnail: "img.png"
---

コンテナのイメージダイジェストはkbldを使えば、Docker CLIも、イメージをダウンロードしなくても調べることができます。<!--more-->

# やり方
[kbldをインストール](https://carvel.dev/kbld/docs/v0.36.0/install/)後、以下のコマンドでできます。

```
echo '{"image": "IMAGE_NAME"}' | kbld --images-annotation=false -f-
```

IMAGE_NAMEには、みたいイメージをいれてください。

# 実行例
nginxのイメージの執筆時点の最新ダイジェストを調べました。

```
% echo '{"image": "nginx:latest"}' | kbld --images-annotation=false -f-
resolve | final: nginx:latest -> index.docker.io/library/nginx@sha256:0047b729188a15da49380d9506d65959cce6d40291ccfb4e039f5dc7efd33286
---
image: index.docker.io/library/nginx@sha256:0047b729188a15da49380d9506d65959cce6d40291ccfb4e039f5dc7efd33286

Succeeded
```
