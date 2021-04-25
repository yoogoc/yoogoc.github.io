---
title: submodule
tags:
- git
date: 2018-11-23 14:10:01
---
### 新增 submodule

```
git submodule add git://github.com/majutsushi/tagbar.git .vim/bundle/tagbar
```

如果這個 repo 先前沒有用過 submodule 那麼 Git 會在目錄下建立一個叫做 .gitmodules 的目錄，這裡記錄了 remote repo 的 URL 和這個 submodule 在此專案的路徑。

執行此命令後 submodule 和 .gitmodules 會自動 staged，這個時候可以 commit 和 push。

### 更新 submodule
個別 repo 更新比較麻煩，必須到個別的目錄底下執行 git pull 去拉 upstream 的程式碼，可是這樣會比較安全；若要一次全部更新所有的 submodule 可以用這 foreach 指令：


```
git submodule foreach --recursive git pull origin master
```

### 刪除 submodule
本以為會有像是 git submodule rm 這樣的指令，結果竟然沒有，必須辛苦地一個一個手動移除，不知道不實作這個指令的考量是什麼，希望未來的版本能把它加上去。

移除 submodule 有以下幾個步驟要做，先把 submodule 目錄從版本控制系統移除：


```
git rm --cached /path/to/files
rm -rf /path/to/files
```

再來是修改 .gitmodules，把不用的 submodule 刪掉，例如：


```
[submodule ".vim/bundle/vim-gitgutter"]
   path = .vim/bundle/vim-gitgutter
   url = git://github.com/airblade/vim-gitgutter.git
-[submodule ".vim/bundle/vim-autoclose"]
-  path = .vim/bundle/vim-autoclose
-  url = git://github.com/Townk/vim-autoclose.git
```

還沒完喔！還要修改 .git/config 的內容，跟 .gitmodules 一樣，把需要移除的 submodule 刪掉，最後再 commit。



### clone 時把 submodule 一起抓下來

執行 git clone 時 Git 不會自動把 submodule 一起 clone 過來，必須加上 --recursive 遞歸參數，這樣可以連帶 submodule 的 submodule 通通一起抓下來：

git clone --recursive git@github.com:chinghanho/.dotfiles.git
如果已經抓下來才發現 submodule 是空的，可以用以下指令去抓，init 會在 _.git/config` 下註冊 remote repo 的 URL 和 local path：


```
git submodule init
git submodule update --recursive
```

或是合併成一行 `git submodule update --init --recursive` 也可以，如果 upstream 有人改過 .gitmodules，那 local 端好像也是用這個方法 update。

### 指令解釋
1. `git submodule init`：根據 .gitmodules 的名稱和 URL，將這些資訊註冊到 .git/config 內，可是把 .gitmodules 內不用的 submodule 移除，使用這個指令並沒辦法自動刪除 .git/config 的相關內容，必須手動刪除；
2. `git submodule update`：根據已註冊（也就是 .git/config ）的 submodule 進行更新，例如 clone 遺失的 submodule，也就是上一段講的方法，所以執行這個指令前最好加上 --init；
3. `git submodule sync`：如果 submodule 的 remote URL 有變動，可以在 .gitmodules 修正 URL，然後執行這個指令，便會將 submodule 的 remote URL 更正。


