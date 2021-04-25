---
title: rails定制化导出
tags:
- rails
date: 2019-05-10 15:43:01

---

> 只是一种实现的思路，照抄照搬行不通

## Installation

```ruby
gem 'rubyzip', '>= 1.2.1'
gem 'axlsx', git: 'https://github.com/randym/axlsx.git', ref: 'c8ac844'
gem 'axlsx_rails'
```

# Usage

### controller

```ruby
render :xlsx, template: "members/index.xlsx.axlsx", layout: false if params[:need_export]
```

> 这里有个小坑，一定要`layout: false` ，只是草草实现了功能还没看他的实现

### view

1. 在`index.erb`  `link_to` 到下载：

   ```ruby
   <%= link_to '导出', members_path(request.parameters.merge({need_export: true})), class: 'btn btn-info' %>
   ```

2. 增加模板 `index.xlsx.axlsx`

   ```ruby
   wb = xlsx_package.workbook

   wb.styles do |style|
     heading = style.add_style(b: true)

     wb.add_worksheet(name: "Leads") do |sheet|

       sheet.add_row %w(姓名 性别 所属城市 手机号 手机号2 邮箱 监护人名字 监护人关系 年龄
                       地址 街道 市场渠道分组 备注 证件号 意向球场 销售顾问 创建时间 学员状态), style: heading

       @members.each do |member|
         sheet.add_row [
           member.name,
           tt(member.gender, 'enum.gender'),
           member.region_name,
           member.mobile,
           member.mobile2,
           member.email,
           member.parent_name,
           tt(member.parent_type, 'enum.parent_type'),
           member.age,
           member.address,
           member.district,
           member.ascription_desc,
           member.comment,
           member.credentials_no,
           member.court_name,
           member.follow_name,
           lt(member.created_at),
           member.tag.try(:name)
         ]
       end
     end
   end

   ```



然后就可以啦！Enjoy this！

> 自我感觉导出模板应该也想html，json一样，具有高度定制化，不建议用Nokogiri的方式去爬，反正都是要请求一次，何必再去算一次dom呢

参考链接：
<https://github.com/jasondoggart/inventory>
<https://github.com/straydogstudio/axlsx_rails>
<https://github.com/randym/axlsx>
