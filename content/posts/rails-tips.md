---
title: rails-tips
date: 2018-12-17 14:50:32
tags:
- rails
---
##Hash#dig
ruby 2.3 中引入了`dig`方法:
通过在每个步骤调用dig从给定key中提取嵌套参数。如果中间步骤为nil，则返回nil
❌：
```ruby
... if params[:user] && params[:user][:address] && params[:user][:address][:somewhere_deep]
```
✔️：
```ruby
... if params.dig(:user, :address, :somewhere_deep)
```

## Object#presence_in
如果接收方包含在参数中，则返回接收方，否则返回nil。参数必须是响应#include?的任何对象。
❌：
```ruby
sort_options = [:by_date, :by_title, :by_author]
...
sort = sort_options.include?(params[:sort])
  ? params[:sort]
  : :by_date
# Another option
sort = (sort_options.include?(params[:sort]) && params[:sort]) || :by_date
```
✔️：
```ruby
params[:sort].presence_in(sort_options) || :by_date
```
## crush
```ruby
class Object
  def crush
    self
  end
end

class Array
  def crush
    r = map(&:crush).compact

    r.empty? ? nil : r
  end
end

class Hash
  def crush
    r = each_with_object({}) do |(k, v), h|
      if (_v = v.crush)
        h[k] = _v
      end
    end

    r.empty? ? nil : r
  end
end

```

## preload scope
错误的案例：
```ruby
class Comment < ActiveRecord::Base
  belongs_to :post

  scope :active, -> { where(soft_deleted: false) }
end
class Post < ActiveRecord::Base
  def active_comments
    comments.active
  end
end
```
这样势必会造成`post.active_comments`的时候会`where`，导致N+1
正确的案例：
```ruby
class Post
  has_many :comments
  has_many :active_comments, -> { active }, class_name: "Comment"
end

class Comment
  belongs_to :post
  scope :active, -> { where(soft_deleted: false) }
end

class PostsController
  def index
    @posts = Post.includes(:active_comments)
  end
end
```
