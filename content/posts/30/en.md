---
title: "How to Look Up an Image Digest without the Docker CLI or Downloading the Image"
date: 2023-01-05T21:30:12+09:00
tags: ["carvel", "kbld"]
thumbnail: "img.png"
---

With kbld you can look up a container image digest without the Docker CLI and without downloading the image.<!--more-->

# How to

After [installing kbld](https://carvel.dev/kbld/docs/v0.36.0/install/), the following command does it:

```
echo '{"image": "IMAGE_NAME"}' | kbld --images-annotation=false -f-
```

Put the image you want to inspect in IMAGE_NAME.

# Example

Looking up the latest digest of the nginx image at the time of writing:

```
% echo '{"image": "nginx:latest"}' | kbld --images-annotation=false -f-
resolve | final: nginx:latest -> index.docker.io/library/nginx@sha256:0047b729188a15da49380d9506d65959cce6d40291ccfb4e039f5dc7efd33286
---
image: index.docker.io/library/nginx@sha256:0047b729188a15da49380d9506d65959cce6d40291ccfb4e039f5dc7efd33286

Succeeded
```
