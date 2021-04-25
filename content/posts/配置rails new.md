---
title: 配置rails new
tags:
- rails
date: 2018-11-23 14:10:01
---
## 配置rails new 默认选项

1. 使用 `rails new --help` 查看:

```
Using --skip-bundle --skip-test -d postgresql from /Users/rocky/.railsrc
Usage:
  rails new APP_PATH [options]

Options:
...

Description:
    The 'rails new' command creates a new Rails application with a default
    directory structure and configuration at the path you specify.

    You can specify extra command-line arguments to be used every time
    'rails new' runs in the .railsrc configuration file in your home directory.

    Note that the arguments specified in the .railsrc file don't affect the
    defaults values shown above in this help message.

Example:
    rails new ~/Code/Ruby/weblog

    This generates a skeletal Rails installation in ~/Code/Ruby/weblog.
```

2. 可以发现 Description下有如下描述：

> You can specify extra command-line arguments to be used every time
    'rails new' runs in the .railsrc configuration file in your home directory.

3. 编辑`.railsrc`文件(`vim ~/.railsrc`)，我的配置:

```
--skip bundle
--skip test
-d postgresql
```
