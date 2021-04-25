---
title: go环境配置
tags:
- go
date: 2019-05-24 14:42:00

---

## mac + vs code 版本go环境搭建

1. mac，vscode, go 准备好(官网有现成的安装包，重在配置)

2. 安装code的go拓展

3. 安装依赖包：

   ```zsh
   go get -u -v github.com/nsf/gocode
   go get -u -v github.com/rogpeppe/godef
   go get -u -v github.com/zmb3/gogetdoc
   go get -u -v github.com/golang/lint/golint
   go get -u -v github.com/lukehoban/go-outline
   go get -u -v sourcegraph.com/sqs/goreturns
   go get -u -v golang.org/x/tools/cmd/gorename
   go get -u -v github.com/tpng/gopkgs
   go get -u -v github.com/newhook/go-symbols
   go get -u -v golang.org/x/tools/cmd/guru
   go get -u -v github.com/cweill/gotests/
   # 调试工具
   go get -v -u github.com/peterh/liner github.com/derekparker/delve/cmd/dlv
   ```


