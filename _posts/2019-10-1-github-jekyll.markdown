---
layout:     post
title:      "Github搭建个人博客的总结"
subtitle:   ""
date:       2019-10-1 12:00:00
author:     "Lzz"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Other
---




## 步骤


### 搭建
可以参考[这个]( https://blog.csdn.net/xudailong_blog/article/details/78762262)



### 执行
cd 到 git 仓库目录，如果要本地调试 的话，直接 执行命令行： Jekyll serve --watch， 注意如果出现一下的错的话，就执行  gem install jekyll-paginate，停止运行Jekyll 直接是 ctrl + C,
<br>

在网页上可以按F5 进行刷新修改的内容。
<br>

Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. If you've run Jekyll with `bundle exec`, ensure that you have included the jekyll-paginate gem in your Gemfile as well. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!

<br>
记录一下 Markdown 语法网站 ： https://www.appinn.com/markdown/#html

<br>
自定义域名，可以参考下 ：https://juejin.im/post/5a71a4f9518825733a3105ac


### 测试
直接在浏览器输入 http://localhost:4000 , 这里的测试地址可以在 命令窗口找到。



  

