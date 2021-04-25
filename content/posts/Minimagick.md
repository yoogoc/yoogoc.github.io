---
title: minimagick水印
tags:
- rails
date: 2018-11-23 14:10:01
---
# 在uploader中使用Minimagick对图片进行处理
``include CarrierWave::MiniMagick``


```
class InsurancePolicyUploader < BaseUploader
  process :watermark
  def store_dir
    "uploads/insurance_policy/#{model.id}"
  end

  def watermark
    manipulate! do |img|
      img.combine_options do |i|
        i.fill 'rgba(0,0,0,0.4)'
        #
        i.font "WenQuanYi-Zen-Hei"
        i.gravity "center"
        i.pointsize "150"
        i.draw "rotate 57 text 0,0 '仅限处理车辆事故及违章办理'"
      end
      img
    end
  end

end
```

## 根据代码进行总结：

1. rgba可以快速的将颜色透明化
2. ImageMagick本身并不支持中文的处理
3. ``convert -list font`` 查看i.font支持的字体


## 存在问题：
1. 水印字体大小是死的，不能做到自适应图片大小来添加水印


> 注：一个令我恍然大悟的网站：[rubblewebs](http://www.rubblewebs.co.uk/imagemagick/imagemagick.php#div2)
