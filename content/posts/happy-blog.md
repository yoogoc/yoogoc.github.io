---
title: happy hexo
date: 2018-11-23 15:24:45
updated: 2020-02-09 21:07:20
tags:
- hexo
---
弄了大半天，可算把写的笔记都搬到hexo来了，对比了一下jekyll和hexo，果然还是hexo好用啊。



### 部署到github pages 每次部署都会丢失`Custom domain`

解决:  在博客的 `source` 目录下新建一个 `CNAME` 文件，然后在这个文件中填入你的域名，这样就不会每次发布之后，`gitpage` 里的 `custom domain` 都被重置掉啦。

