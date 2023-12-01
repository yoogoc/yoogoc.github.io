+++
title = "About"
description = "Hugo, the world’s fastest framework for building websites"
date = "2023-12-01"
aliases = ["about-us","about-hugo","contact"]
author = "Hugo Authors"
+++

1996年出生，2019年结婚，同年有了一个女儿。

喜欢计算机科学，对各种编程语言、前端后端客户端移动端都感兴趣。

目前工作于q2bi.com,用go写后端。

微信号：yoogoc

我的.gitconfig

```
[user]
  name = yoogo
  email = yoogoc@163.com
[pull]
  rebase = false
[init]
  defaultBranch = main
[push]
  default = current
[rebase]
  abbreviateCommands = true
[alias]
  a = add
  co = checkout
  cm = commit -m
  dfs = diff --staged
  s = status
  r = reset
  df = diff
  b = branch
  lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
  cp = cherry-pick
  c = checkout
  caane = commit -a --amend --no-edit
[core]
  excludesfile = ~/.gitignore_global
  ignorecase = false
[log]
	date = iso
[merge]
  conflictstyle = diff3
[diff]
  # external = difft

[http "https://github.com"]
  proxy = http://127.0.0.1:7890
[http "https://go.googlesource.com"]
  proxy = http://127.0.0.1:7890
[http "https://sourceware.org"]
  proxy = http://127.0.0.1:7890
# [http "https://gcc.gnu.org"]
#   proxy = http://127.0.0.1:7890
[http "https://git.musl-libc.org"]
  proxy = http://127.0.0.1:7890

[filter "lfs"]
	clean = git-lfs clean -- %f
	smudge = git-lfs smudge -- %f
	process = git-lfs filter-process
	required = true
```
