---
layout: default
title: web.py 项目架构分析之 googlemodules
category: 源代码阅读
comments: true
---

# web.py 项目之 googlemodules

## 项目说明

项目来自 [webpy.org](http://webpy.org/src/), 这是一个真实的在线上运行的
项目: [Google Modules](http://www.googlemodules.com/), 可以上传，下载，
一些模块，还有一些评分，打标签等等功能。(不过这网站挺奇怪的。。。)

## 目录树

```
src/
    ├── application.py
    ├── forum.py
    ├── config_example.py
    ├── INSTALL
    ├── LICENCE
    ├── app
    │   ├── controllers
    │   ├── helpers
    │   ├── models
    │   └── views
    ├── app_forum
    │   ├── controllers
    │   ├── models
    │   └── views
    ├── data
    ├── public
    │   ├── css
    │   ├── img
    │   │   └── star
    │   ├── js
    │   └── rss
    ├── scripts
    └── sql

18 directories
```

终于遇到个稍微大一点的项目了，要好好看看。

从目录上看，整个项目分成两个部分，app 和 app_forum,每个部分都使用了
典型的MVC结构，将app分成 controllers, models, views 三大部分。

另外，网站使用的 css, js 文件，图片，也都统一放在了public目录下。

INSTALL 文件描述了如何安装部署项目, 包括在哪里下载项目，哪里下载web.py，如何
配置 lighttpd, 如何配置项目。

config_example.py 文件给了一个配置文件模板，按自己的需要修改其中内容，最后
把文件名改为 config.py 就可以了，其中包括对数据的配置，调试，缓存的开启等等。

LICENCE 文件描述了项目使用的开源协议: GPLv3。

项目使用的脚本放在scripts目录下，创建数据库使用的文件放在了sql目录下。

## 代码统计

先看看代码统计

![googlemodules_code_stat.jpg](/assets/blog-images/googlemodules_code_stat.jpg)

## application 模块

### application.py

```python

#!/usr/bin/env python
# Author: Alex Ksikes 

# TODO: 
# - setup SPF for sendmail and 
# - emailerrors should be sent from same domain
# - clean up schema.sql
# - because of a bug in webpy unicode search fails (see models/sql_search.py)

import web
import config
import app.controllers

from app.helpers import custom_error

import forum

urls = (        
    # front page
    '/',                                    'app.controllers.base.index',
    '/page/([0-9]+)/',                      'app.controllers.base.list',
    
    # view, add a comment, vote
    '/module/([0-9]+)/',                    'app.controllers.module.show',
    '/module/([0-9]+)/comment/',            'app.controllers.module.comment',
    '/module/([0-9]+)/vote/',               'app.controllers.module.vote',
    
    # submit a module
    '/submit/',                             'app.controllers.submit.submit',
    
    # view author page
    '/author/(.*?)/',                       'app.controllers.author.show',      
    
    # search browse by tag name
    '/search/',                             'app.controllers.search.search',    
    '/tag/(.*?)/',                          'app.controllers.search.list_by_tag',
    
    # view tag clouds
    '/tags/',                               'app.controllers.cloud.tag_cloud',
    '/authors/',                            'app.controllers.cloud.author_cloud',
    
    # table modules
    '/modules/(?:by-(.*?)/)?([0-9]+)?/?',   'app.controllers.all_modules.list_by',
    
    # static pages
    '/feedback/',                           'app.controllers.feedback.send',
    '/about/',                              'app.controllers.base.about',
    '/help/',                               'app.controllers.base.help',
    
    # let lighttpd handle in production
    '/(?:css|img|js|rss)/.+',               'app.controllers.public.public',
    
    # canonicalize /urls to /urls/
    '/(.*[^/])',                            'app.controllers.public.redirect',

    # mini forum app
    '/forum',                               forum.app,    

    '/hello/(.*)',                            'hello',
    
    # site admin app
#    '/admin',                              admin.app,    
)

app = web.application(urls, globals())
custom_error.add(app)

if __name__ == "__main__":
    app.run()
```

可以看出，这是 application 部分的入口，这个模块仅仅是定义了各个请求的处理方式，
并完成程序的启动，所有的实现均不在这里出现，而是通过 `import` 导入，特别需要
注意 `urls` 最后定义的 `/forum` 和 `/admin` 使用了子程序，而不是通过之前的字符串
实现映射。还需要注意对静态文件,即css,js,img,rss文件的单独处理。

所有这些都与之前分析过的那些小项目不同，回想起我之前写的 
[BlogSystem](https://github.com/minixalpha/BlogSystem), 所有的处理实现都放在
同一个文件中，导致最后一个文件居然 700多行，真是让人潸然泪下。。。
而且之前也不知道使用子程序，所有处理都堆在一起。看来读完这份源代码，真应该重构一
下了。

### app 模块

```
app/
    ├── models # 数据模块，MVC中的 M
    ├── views # 显示模块，MVC中的 V
    ├── controllers # 控制模块，MVC中的 C
    └── helpers # 辅助模块，实现辅助功能

4 directories
```

### controllers 模块

```
controllers/
    ├── base.py         # 对基本页面，如网站主页,关于网页，帮助等的处理
    ├── all_modules.py  # 显示全部模块
    ├── module.py       # 对模块页面的处理
    ├── search.py       # 对搜索模块功能处理
    ├── submit.py       # 对提交模块的处理
    ├── author.py       # 查看模块作者信息
    ├── cloud.py        # 对标签云页面进行处理
    ├── feedback.py     # 处理反馈信息
    └── public.py       # 对静态文件的处理
```

这个模块主要是对请求处理的实现，在 `urls` 里定义的那些映射关系，
很多被映射到这里。

实现过程中，调用 models 模块对数据操作，再送入 views 模块通过模板引擎显示数据内容。

### models 模块

```
models/
    ├── comments.py     # 对评论数据的处理
    ├── modules.py      # 对模块数据的处理
    ├── rss.py          # 对 rss 订阅的处理
    ├── sql_search.py   # 对搜索的数据处理
    ├── submission.py   # 对用户提交内容的处理
    ├── tags.py         # 对标签内容的数据处理
    └── votes.py        # 对用户投票的数据处理
```

这个模块直接调用 web.py 的db模块对数据库进行操作，对数据库的连接在 config.py 中
已经完成。这里完成数据的获取，处理，返回。可以看出，对不同种类的数据又分成了
很多小的模块。


### views 模块

```
views/
    ├── about.html
    ├── all_modules.html
    ├── faq.html
    ├── feedback.html
    ├── help.html
    ├── internal_error.html
    ├── layout.html
    ├── list_modules_by_author.html
    ├── list_modules.html
    ├── not_found.html
    ├── rss.html
    ├── show_module.html
    ├── submit_module.html
    ├── submitted_form.html
    ├── tag_cloud_author.html
    └── tag_cloud.html
```

这个模块是各个页面的模板文件所在，通过模板引擎，这些模板的内容
被填充，最终返回给用户。


### helpers 模块

```
helpers/
    ├── custom_error.py     # 错误处理，如未找到页面等等
    ├── formatting.py       # 格式处理相关的工具函数
    ├── image.py            # 图片处理相关函数
    ├── misc.py             # 用于生成一些随机数据
    ├── paging.py           # 用于多条信息分成多个页面
    ├── render.py           # 对基本布局的包装
    ├── strip_html.py       # HTML 文件的处理
    ├── tag_cloud.py        # 对标签云的处理
    └── utils.py            # 常用的小工具函数
```

这一模块把需要用到的一些工具类函数分成多个类组织在一起，
这样可以避免之前的 MVC 模块太臃肿。


## forum 模块

在 `application.py` 中定义了对论坛请求的处理，这是在子程序中进行处理的。

```
    # mini forum app
    '/forum',                               forum.app,    
```

对 `/forum` 的请求会发到子程序 `forum.py`, 这是一个类似 `application.py`
的程序：

```python
#!/usr/bin/env python
# Author: Alex Ksikes 

import web
import config
import app_forum.controllers

from app.helpers import custom_error

urls = (
    '/',                             'app_forum.controllers.base.index',
    '/new/',                         'app_forum.controllers.base.new',
    '/page/([0-9]+)/',               'app_forum.controllers.base.list',
    '/thread/([0-9]+)/',             'app_forum.controllers.base.show',
    '/thread/([0-9]+)/reply/',       'app_forum.controllers.base.reply',
)

app = web.application(urls, globals())
custom_error.add(app)

if __name__ == "__main__":
    app.run()
```

可以看出子程序的作用。在 application 中没有定义　`/forum/new/` ，但是对
`/forum` 的请求会发给这里的 `forum.py`, 这个程序定义了对 `/new` 的处理，
它们连起来，构成了对 `/forum/new/` 处理的映射。


forum 模块的结构与 application 模块是相似的:

```
app_forum/
    ├── controllers
    │   ├── base.py         # 对基本页面，如论坛主页的处理
    ├── models
    │   └── threads.py      # 对论坛数据的处理
    └── views               # 论坛页面模板
        ├── list_threads.html
        ├── show_thread.html
        └── submitted_form.html
```

可以看出，只有很少的文件, 只搭了个框架，具体的内容我们就不分析了。
