---
layout: default
title: web.py 项目架构分析之 todolist
category: 源代码阅读
comments: true
---

# web.py 项目之 todolist

## 目录树

```
.
└── src
    ├── model.py
    ├── schema.sql
    ├── templates
    │   ├── base.html
    │   └── index.html
    ├── todo.py

2 directories, 8 files
```

## 项目说明

项目来自 [webpy.org](http://webpy.org/src/todo-list/0.3),
主要实现一个基于web的 todolist，就是可以添加删除一些任务。
非常简单

## 结构分析

### 控制模块

控制模块只有todo.py

```python

""" Basic todo list using webpy 0.3 """
import web
import model

### Url mappings

urls = (
    '/', 'Index',
    '/del/(\d+)', 'Delete'
)


### Templates
render = web.template.render('templates', base='base')


class Index:

    form = web.form.Form(
        web.form.Textbox('title', web.form.notnull, 
            description="I need to:"),
        web.form.Button('Add todo'),
    )

    def GET(self):
        """ Show page """
        todos = model.get_todos()
        form = self.form()
        return render.index(todos, form)

    def POST(self):
        """ Add new entry """
        form = self.form()
        if not form.validates():
            todos = model.get_todos()
            return render.index(todos, form)
        model.new_todo(form.d.title)
        raise web.seeother('/')



class Delete:

    def POST(self, id):
        """ Delete based on ID """
        id = int(id)
        model.del_todo(id)
        raise web.seeother('/')


app = web.application(urls, globals())

if __name__ == '__main__':
    app.run()
```

这一模块是对页面逻辑的处理，包括：

* 处理页面请求 Index.GET
* 处理添加请求 Index.POST
* 处理删除请求 Delete.POST

    这是MVC中的C吗


### 显示模块

显示模块是 templates 下的两个文件，base.html, index.html
框架使用模板引擎已经将显示与实现分开了，这个项目没有像
skeleton 项目那样有专门的一个 view.py 提供具体的与显示相关
的逻辑。

templates/base.html

```html

$def with (page)

<html>
<head>
    <title>Todo list</title>
</head>
<body>

$:page

</body>
</html>
```

templates/index.html

```html

$def with (todos, form)

<table>
    <tr>
        <th>What to do ?</th>
        <th></th>
    </tr>
$for todo in todos:
    <tr>
        <td>$todo.title</td>
        <td>
            <form action="/del/$todo.id" method="post">
                <input type="submit" value="Delete"/>
            </form>
        </td>
    </tr>    
</table>  

<form action="" method="post">
$:form.render()
</form>
```

注意这里表单的实现，是在控制模块中生成了 form, 传给模板引擎。
我觉得还是应该分开好。毕竟 form 只与显示相关，和控制无关。

### 数据模块

数据模板提供了 model.py 处理数据库操作，数据库操作并不直接在
控制模块中体现，而是由 model.py 提供接口。另外还包括 schema.sql
提供数据库表格式。

schema.sql

```
CREATE TABLE todo (
    id INT AUTO_INCREMENT,
    title TEXT,
    primary key (id)
);
```

model.py

```python
import web

db = web.database(dbn='mysql', db='todo', user='root', pw="XXX")

def get_todos():
    return db.select('todo', order='id')

def new_todo(text):
    db.insert('todo', title=text)

def del_todo(id):
    db.delete('todo', where="id=$id", vars=locals())
```

总的来说，项目还是太小了，无法体会到当初作 BlogSystem 时遇到的困难应该怎么
解决，不过MVC的结构已经非常明显了，希望以后可以阅读一些大型网站的结构。
