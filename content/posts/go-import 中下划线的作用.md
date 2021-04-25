---
title: go-import 中下划线的作用
tags:
- go
date: 2018-11-23 14:10:01
---
###  在Golang里，import的作用是导入其他package，但是今天在看beego框架时看到了import 下划线，不知其意，故百度而解之。

 　　import 下划线（如：import _ hello/imp）的作用：当导入一个包时，该包下的文件里所有init()函数都会被执行，然而，有些时候我们并不需要把整个包都导入进来，仅仅是是希望它执行init()函数而已。这个时候就可以使用 import _ 引用该包。**即使用【import _ 包路径】只是引用该包，仅仅是为了调用init()函数，所以无法通过包名来调用包中的其他函数。**
