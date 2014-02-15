---
layout: default
title: web.py 源代码分析之 web.test.application.test_UppercaseMethods
category: 源代码阅读
comments: true
---

# 分模块测试

## application.py

对 application.py 的测试，调用命令：

    python test/application.py


### test_UppercaseMethods(self)

这个函数源代码是：

```python
def testUppercaseMethods(self):
    urls = ("/", "hello")
    app = web.application(urls, locals())
    class hello:
        def GET(self): return "hello"
        def internal(self): return "secret"
        
    response = app.request('/', method='internal')
    self.assertEquals(response.status, '405 Method Not Allowed')
```

这个函数似乎是在测试 `request` 请求的方法不正确时，`application`
的反应。

我们打印一下 `response` 的值，发现内容是这样的：

    <Storage {'status': '405 Method Not Allowed', 
    'headers': {'Content-Type': 'text/html', 
    'Allow': 'GET, HEAD, POST, PUT, DELETE'}, 
    'header_items': [('Content-Type', 'text/html'), 
    ('Allow', 'GET, HEAD, POST, PUT, DELETE')], 'data': 'None'}>

从这里可以看出允许的方法：

    'Allow': 'GET, HEAD, POST, PUT, DELETE'

`response.status` 最终值为：

    'status': '405 Method Not Allowed',


我们再一次深入 `web.application.request`, 看一看具体实现。

```python
    def request(self, localpart='/', method='GET', data=None,
                host="0.0.0.0:8080", headers=None, https=False, **kw):
        
        ...

        env = dict(
            env, 
            HTTP_HOST=host, 
            REQUEST_METHOD=method, 
            PATH_INFO=path, 
            QUERY_STRING=query, 
            HTTPS=str(https)
            )

        ...

        response = web.storage()
        def start_response(status, headers):
            response.status = status
            response.headers = dict(headers)
            response.header_items = headers

        response.data = "".join(self.wsgifunc()(env, start_response))

        return response
```

我们只保留了关键代码，可以看出，method 首先被设置在了 `env` 中，然后，
通过 

```python
    self.wsgifunc()(env, start_response))
```

`start_response` 被 `wsgifunc()` 返回的函数调用，`response` 的值被设置。 如果阅读了前面推荐的几篇关于 `WSGI` 的文章，对这里的 `self.wsgifunc()` 和 `start_response`应该不会感觉到陌生。

好，我们继续深入 `web.application.wsgifunc()`

```python
    def wsgifunc(self, *middleware):
        
        ...

        def wsgi(env, start_resp):
            # clear threadlocal to avoid inteference of previous requests
            self._cleanup()

            self.load(env)
            try:
                # allow uppercase methods only
                if web.ctx.method.upper() != web.ctx.method:
                    raise web.nomethod()

                result = self.handle_with_processors()
                if is_generator(result):
                    result = peep(result)
                else:
                    result = [result]
            except web.HTTPError, e:
                result = [e.data]


            result = web.safestr(iter(result))

            status, headers = web.ctx.status, web.ctx.headers
            start_resp(status, headers)
            
            def cleanup():
                self._cleanup()
                yield '' # force this function to be a generator

            return itertools.chain(result, cleanup())

        for m in middleware: 
            wsgi = m(wsgi)

        return wsgi
```

从上述代码可以看出，返回的函数就是 `wsgi`, `wsgi` 在返回前，可能
会使用 `middleware` 进行一层一层的包装。不过这里，我们只看 `wsgi`
的代码就可以了。

关键的两行在：

```python
status, headers = web.ctx.status, web.ctx.headers
start_resp(status, headers)
```

这里，`web.application.requests.start_response` 函数 被调用，并设置好
`status` 和 `headers`，二者的值从 `web.ctx` 中取得。所以，下一部，我们
关心 `web.ctx` 值的来源。因为 `method` 的信息在 `env` 中，所以 
`web.ctx` 的值一定会受 `env` 的影响，所以在 `wsgi` 中要找到 `env`是如何
被使用的。

```python
self.load(env)
try:
    # allow uppercase methods only
    if web.ctx.method.upper() != web.ctx.method:
        raise web.nomethod()
```

我们猜想 `self.load(env)` 对 `web.ctx` 进行了设置，然后后面的 `try`
对 `web.ctx` 检查，这里对 `upper`的检查，让我们想到了，原来一开始的
`testUppercaseMethods` 是在测试 `method` 名字是不是全是大写。。。

于是在 `raise web.nomethod()` 前加一个 `print` ，测试一下是不是执行到
了这里，发现还真是。那么我们再深入 `load` ，看看如何根据 `env` 设置
`web.ctx`。

```python
def load(self, env):
    """Initializes ctx using env."""
    ctx = web.ctx
    ctx.clear()
    ctx.status = '200 OK'

    ...

    ctx.method = env.get('REQUEST_METHOD')

    ...
```

`load` 函数会根据 `env` 设置 `web.ctx` ，但是只是简单赋值，没有对
`method` 方法的检查。于是，我明白了，原来是在 `raise web.nomethod()`中对
`status` 进行了设置。

在 `raise` 之前打印 `web.ctx.status` 的值，果然是 `200 OK`。那我们再进入
`web.nomethod` 看看吧。

这个函数在 `webapi.py` 中，进入文件一看，果然是对许多状态的定义。
找到 `nomethod` ，发现指向 `NoMethod`，下面看 `NoMethod`:

```python
class NoMethod(HTTPError):
    """A `405 Method Not Allowed` error."""
    def __init__(self, cls=None):
        status = '405 Method Not Allowed'
        headers = {}
        headers['Content-Type'] = 'text/html'
        
        methods = ['GET', 'HEAD', 'POST', 'PUT', 'DELETE']
        if cls:
            methods = [method for method in methods if hasattr(cls, method)]

        headers['Allow'] = ', '.join(methods)
        data = None
        HTTPError.__init__(self, status, headers, data)
```

发现还真是对错误号 `405` 的处理。 设置好status, 最终的处理
发给了 `HTTPError`，所以我们再进入 `HTTPError`。

```python
class HTTPError(Exception):
    def __init__(self, status, headers={}, data=""):
        ctx.status = status
        for k, v in headers.items():
            header(k, v)
        self.data = data
        Exception.__init__(self, status)
```

这里终于发现了对 `ctx.status` 的设置。这就是在这里我们知道了下面这两
行关键代码的最终来源。

```python
status, headers = web.ctx.status, web.ctx.headers
start_resp(status, headers)
```

前者设置 `status` 和 `headers`，后者调用 `requests` 函数中的 `start_response` 对
 `response` 的状态进行了设置。最后返回给最一开始 `testUppercaseMethods` 中的 
 `response`。


