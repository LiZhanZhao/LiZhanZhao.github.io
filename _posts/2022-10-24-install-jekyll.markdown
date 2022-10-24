---
layout:     post
title:      "安装 jekyll"
subtitle:   ""
date:       2022-10-24 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---


## 参考
[这里](https://www.jekyll.com.cn/docs/installation/windows/)

## 总结步骤
- 下载 [ RubyInstaller Downloads](https://www.jekyll.com.cn/docs/installation/windows/)
<br>
<br>
- 执行 gem install jekyll bundler 命令
<br>
<br>
- 执行 gem install jekyll-paginate 命令，不执行这一个可能会有 "Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. If you've run Jekyll with `bundle exec`, ensure that you have included the jekyll-paginate gem in your Gemfile as well. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!"
<br>
<br>
- 运行 jekyll serve --watch 命令就可以本地查看网页效果了