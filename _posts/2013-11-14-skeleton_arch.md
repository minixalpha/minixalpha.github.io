---
layout: default
title: web.py 项目架构分析之 skeleton 
category: 源代码阅读
comments: true
---

# web.py 项目之 skeleton

skeleton 是 web.py 官网上给出的一个最简单的项目结构示例。

## 目录树
```
.
└── src  
    ├── code.py  
    ├── config.py  
    ├── db.py  
    ├── sql
    │   └── tables.sql
    ├── templates
    │   ├── base.html
    │   ├── item.html
    │   └── listing.html
    └── view.py

3 directories, 9 files
```

## 结构分析

### 控制模块

```python
#code.py

import web
import view, config
from view import render

urls = (
    '/', 'index'
)

class index:
    def GET(self):
        return render.base(view.listing())

if __name__ == "__main__":
    app = web.application(urls, globals())
    app.internalerror = web.debugerror
    app.run()
```

`code` 模块作为入口:

* app 的创建与启动
* url　与　处理类的映射与处理入口

但是，具体的处理并不在这里实现。而是放在了 `view` 模块中。

    这一模块是MVC中的C吗？

### 显示模块

```python
#view.py

import web
import db
import config

t_globals = dict(
  datestr=web.datestr,
)
render = web.template.render('templates/', cache=config.cache, 
    globals=t_globals)
render._keywords['globals']['render'] = render

def listing(\**k):
    l = db.listing(\**k)
    return render.listing(l)
```

从这里可以看出　`view` 模块

* 与模板关联 
* 从数据库中取数据，然后发给模板

我们再来看看模板:

```html
<!--
    templates/base.html
-->

$def with (page, title=None)
<html><head>
<title>my site\
$if title: : $title\
</title>
</head><body>
<h1><a href="/">my site</a></h1>
$:page   
</body></html>
```

`base.html` 是所有模块的公共部分，每个模块只需要提供
它的 `page` ，即内容就可以了。

```html
<!--
    templates/listing.html
-->

$def with (items)

$for item in items:
    $:render.item(item)

```

    这一模块是MVC中的V吗

### 数据操作

数据库操作分三部分

* sql/tables.sql 数据库表定义
* config.py 数据库连接
* db.py 数据库操作

```
/* tables.sql */
CREATE TABLE items (
    id serial primary key,
    author_id int references users,
    body text,
    created timestamp default current_timestamp 
);
```

```
#config.py

import web
DB = web.database(dbn='mysql', db='skeleton', user='root', pw='xx')
cache = False
```

```
# db.py
import config

def listing(\**k):
    return config.DB.select('items', **k)
```

    这是MVC中的M吗

这是 web.py 中最基本的一个项目结构。应该是使用的MVC设计模式。但是
因为程序本身不大，还体会不到 MVC 的好处。
