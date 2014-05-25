此Git库为我的博客[一棵波菜](http://yikebocai.com)的源码, fork自 [Luke's Homepage](http://geeklu.com)。
使用Jekyll进行搭建，Jekyll是一个Ruby写的程序，可以将Markdown写的文章通过模板生成最终的Html静态文件。

## 更新纪要

### 2014-05-25

终于把表格的问题搞定了，其实在`_config.yml`中配置的信息并没有问题,只要增加`redcarpet`的扩展`tables`即可，原来表格没有自动转换的原因经过反复实验发现是表格左边没有紧挨边框前面还有个空格的原因，折腾了我好久。另外原来的css模板并没有增加表格的样式，参考bootstrap的风格设置了表格样式，终于看起来比较舒服了。另外原来的时间默认不是本地时间，需要在配置文件中设置一下`timezone`为本地时区，配置如下：

```
permalink: /:year/:month/:title/
url: http://yikebocai.com
author: yikebocai
timezone: Asia/Shanghai
paginate: 20
markdown: redcarpet
redcarpet:
  extensions: ["fenced_code_blocks", "tables", "highlight", "with_toc_data", "strikethrough", "underline"]
```
