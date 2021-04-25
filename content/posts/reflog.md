---
title: git有用的操作
tags:
- git
date: 2018-11-23 14:10:01
---
### `git reflog`

- 当你提交或修改分支时，reflog 就会更新。
- 任何时间运行 git reflog 命令可以查看当前的状态

当你的commit是追加到上一次的commit，例如 `git commit -a --amend` ，这时，意识到错误了，想要撤销这一次的commit，且不伤害原有的commit, 可以有如下操作：

```bash
git reflog  #查看自己的commit，pull,rebase,reset,checkout 等操作
git reset HEAD@{?}  #从reflog拿出你想还原到什么位置

```



### `git cat-file -t xxx`

- xxx是随便的字符串，cat-file -t  可以判断xxx的类型
- cat-file -p 查看xxx内容



###  `git branch -av`



### `git branch --set-upstream-to=aaaaa/master vvvvv`

给 aaaaa remote设置默认分支为 vvvvv



### git repo同步

git clone 虽然会把所有git库拉下来，但是并不会在本地库创建remote的分支

为了解决这个问题，因此有如下方案

1. 普通clone，拉下来后脚本创建branch：

```
git clone xxx.git
cd xxx
for branch in `git branch -r|grep -v ' -> '|cut -d"/" -f2`; do git checkout $branch; git fetch; done;
```

2. 镜像方式clone，拉完后手动还原git repo

```
mkdir my_repo_folder && cd my_repo_folder
git clone --mirror xxx.git .git
git config --bool core.bare false
git reset --hard
```

