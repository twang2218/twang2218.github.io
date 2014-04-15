准备环境
==========

```bash
git clone git@github.com:twang2218/twang2218.github.io.git
cd twang2218.github.io.git
git checkout dev
git submodule update --init --recursive
```

本地服务
=========

- 普通模式

```bash
hexo server
```

- 带草稿模式

```bash
hexo server -d
```

部署
========

- github

```bash
hexo clean
hexo deploy
```