---
title: 在 Debian 12 的 docker image 上無法 apt-get update 的問題
date: 2023-07-17 18:26:16
tags: Docker, Debian, 筆記
---

最近在一台稍老的機器上要使用 Debian 12 的 image 來 build 時，不管是執行 `apt update` 或 `apt-get update` 都會產生類似以下的錯誤

```
E: Problem executing scripts APT::Update::Post-Invoke 'rm -f /var/cache/apt/archives/.deb /var/cache/apt/archives/partial/.deb /var/cache/apt/*.bin || true'
```

後來在 stackoverflow 上找到[這篇文章](https://stackoverflow.com/questions/72624687/apt-get-update-fails-on-ubuntu-22-base-docker-image)，看起來結論是新版的 Docker 有更新了一些 syscall，如果使用了舊版的 Docker 就有可能產生這個問題。

這次產生問題的機器原先是使用 Ubuntu 20.04.6 + Docker 20.04 會產生此問題，更新到最新的 Docker 24.0.4 就解決了。