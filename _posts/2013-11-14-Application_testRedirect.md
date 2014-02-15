---
layout: default
title: web.py 源代码分析之 web.test.application.testRedirect
category: 源代码阅读
comments: true
---

# 分模块测试

## application.py

对 application.py 的测试，调用命令：

    python test/application.py


### testRedirect

```python
def testRedirect(self):
    urls = (
        "/a", "redirect /hello/",
        "/b/(.*)", r"redirect /hello/\1",
        "/hello/(.*)", "hello"
    )
    app = web.application(urls, locals())
    class hello:
        def GET(self, name): 
            name = name or 'world'
            return "hello " + name
        
    response = app.request('/a')
    self.assertEquals(response.status, '301 Moved Permanently')
    self.assertEquals(
        response.headers['Location'], 
        'http://0.0.0.0:8080/hello/')

    response = app.request('/a?x=2')
    self.assertEquals(response.status, '301 Moved Permanently')
    self.assertEquals(
        response.headers['Location'], 
        'http://0.0.0.0:8080/hello/?x=2')

    response = app.request('/b/foo?x=2')
    self.assertEquals(response.status, '301 Moved Permanently')
    self.assertEquals(
        response.headers['Location'], 
        'http://0.0.0.0:8080/hello/foo?x=2')
```

看到这段代码首先对 `urls` 挺好奇的，`urls` 一般是一个 `url` 对应
一个处理它的类，可是 `redirect /hello/` 是什么意思？所以，我们
有必要看一下 `web.application` 如何对 `urls` 进行处理。

我们还从这句开始：

```python
response = app.request('/a')
```

看看 `urls` 是如何被处理的。

```python
# web.application.request

def request(self, localpart='/', method='GET', data=None,
            host="0.0.0.0:8080", headers=None, https=False, **kw):


    path, maybe_query = urllib.splitquery(localpart)
    query = maybe_query or ""

    ...

    env = dict(env, HTTP_HOST=host, REQUEST_METHOD=method, PATH_INFO=path, 
        QUERY_STRING=query, HTTPS=str(https))

    ...

    response.data = "".join(self.wsgifunc()(env, start_response))

```

可以看出，请求的 `url` 被分成两部分： `path` 和 `maybe_query`, 
然后传入 `env`中。

`urllib` 是标准库的一部分，但是在文档中没有对 `splitquery`
有说明，这可能是一个非公开的API。通过

    >>> help(urllib.splitquery)

可以得到

    splitquery('/path?query') --> '/path', 'query'

看来这个调用是把 `url` 请求分成路径与请求两个部分，这也和
返回结果的赋值保持一致。

另外，最好不要使用这个函数，python 2.7 中提供了 `urlparse`
模块，可以完成同样功能（甚至更多），python 3 中这个模块
更改为 `urllib.parse`。

这里不再多说，只是明白这一句是要做什么就好。我们继续看数据
封装在 `env` 后发生的故事。

我们又来到这里：

```python
    response.data = "".join(self.wsgifunc()(env, start_response))
```

最终对 `env` 的处理在 `wsgi` 函数中。

```python
# web.application.wsgifunc.wsgi

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
```

同样，先把 `env` 载入 `web.ctx`， 然后我们通过 `print` 定位到
这一句改变了 `web.ctx.status` 的值。 

```
                result = self.handle_with_processors()
```

可见这里对 `url` 进行了分析。下面我们深入下去。

```python
# web.application.handle_with_processors

def handle_with_processors(self):
    def process(processors):
        try:
            if processors:
                p, processors = processors[0], processors[1:]
                return p(lambda: process(processors))
            else:
                return self.handle()
        except web.HTTPError:
            raise
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            print >> web.debug, traceback.format_exc()
            raise self.internalerror()
    
    # processors must be applied in the resvere order. (??)
    return process(self.processors)
```

这里 `processors` 为空，所以进入 `self.handle()`

```python
# web.application.handle

def handle(self):
    fn, args = self._match(self.mapping, web.ctx.path)
    return self._delegate(fn, self.fvars, args)
```

这个 `self.mapping` 是将我们的 `urls` 转化成两两一组后的列表。
先看看 `_match`函数。

```python
# web.application._match

def _match(self, mapping, value):
    for pat, what in mapping:
        if isinstance(what, application):
            if value.startswith(pat):
                f = lambda: self._delegate_sub_application(pat, what)
                return f, None
            else:
                continue
        elif isinstance(what, basestring):
            what, result = utils.re_subm('^' + pat + '$', what, value)
        else:
            result = utils.re_compile('^' + pat + '$').match(value)
            
        if result: # it''s a match
            return what, [x for x in result.groups()]
    return None, None
```

可以看出这是一个循环，根据当前的 `value` ，即 `web.ctx.path`
去查找 `urls` 中定义的对应项。what就是这个项。
先使用 `isinstance(what, application)` 看是不是使用了子程序。
再看看what是不是 `basestring` 的实例。直到运行至
result不为空，这说明找到了一个匹配。然后会返回匹配中
的所有分组。

当前的运行会选择第二个分支。
即：

```python
# web.application._match

elif isinstance(what, basestring):
    what, result = utils.re_subm('^' + pat + '$', what, value)
```

`utils.re_subm` 对路径中的正则表达式进行处理。`pat` 和 `what`
是 `urls` 中对应的项， `value` 是当前的请求路径。

```python
# utils.re_subm

def re_subm(pat, repl, string):
    """
    Like re.sub, but returns the replacement _and_ the match object.
    
        >>> t, m = re_subm('g(oo+)fball', r'f\\1lish', 'goooooofball')
        >>> t
        'foooooolish'
        >>> m.groups()
        ('oooooo',)
    """
    compiled_pat = re_compile(pat)
    proxy = _re_subm_proxy()
    compiled_pat.sub(proxy.__call__, string)
    return compiled_pat.sub(repl, string), proxy.match

```

这里 `re_compile(pat)` 的含义与 `re.complie(pat)` 类似，
返回一个`RegexObject` 对象，只不过加入了 `Cache` 机制，避免多次执行 `re.complie` 调用。

下面看这两行代码

```python
proxy = _re_subm_proxy()
compiled_pat.sub(proxy.__call__, string)
```

其中 `_re_subm_proxy` 定义为：

```python
class _re_subm_proxy:
    def __init__(self): 
        self.match = None
    def __call__(self, match): 
        self.match = match
        return ''
```

`compiled_pat` 会与 `string` 一起生成一个 `Match` 对象，
这个对象会存储在一个 `_re_subm_proxy` 对象，即 `proxy`中
我们可以看到 `return` 中，`proxy` 最后会将其 `match` 返回。
我在想，为什么使用`search`直接生成一个 `Match`  对象然后返回呢？
查了一下，这似乎与 `代理模式` 相关。但具体为什么还不知道。又查了很久，似乎又与 `弱引用`, 和之前的 Cache 相关,
但是不确定。

    问题：为什么要使用代理类

最后的 `return` 语句返回两个值，其中
`compiled_pat.sub (repl, string)` 是把 `string` 与 `pat`
中匹配的部分，用于替换 `repl` 中对应的组号。`proxy.match` 就是 `pat` 和 `string` 匹配得到的　`Mathc`对象。

关于 `Python 正则表达式` 可以参考这篇：
[Python 正则表达式指南](http://www.cnblogs.com/huxi/archive/2010/07/04/1771073.html)

到这里，我们可能就明白了一开始的时候的那段：

```python
# web.application._match

elif isinstance(what, basestring):
    what, result = utils.re_subm('^' + pat + '$', what, value)
```

的含义，它的意思是如何 `urls` 里有一对 `(url,class)`
其中 `url` 和　`class` 都是用正则表达式表示的，
这时候实际来了一个请求 `r_url`，它会与 `url`进行
匹配，根据这个匹配生成相应的 `r_class`。

看看刚才的示例就明白了：


    >>> t, m = re_subm('g(oo+)fball', r'f\\1lish', 'goooooofball')
    >>> t
    'foooooolish'

你可以把 `re_subm` 的三个参数依序当成 `url`, `class`，
以及 `r_url`，最后得到的 `t` 就是 `r_class`。

举个例子，假如有一系列页面，分别是 `req1`, `req2`, 
`req3`, ... , `reqN`, 需要处理。

那么就可以在 `urls` 里加入

    (r'req(\d+)', r'proc\1')

这样，如果来了请求　`req2`，通过`re_subm`自然会解析成 `proc2`。

    >>> t, m = re_subm(r'req(\d+)', r'proc\1', 'req2')
    >>> t
    'proc2'
    >>> m.group(0)
    'req2'
    >>> m.group(1)
    '2'

总之，

```python
def handle(self):
    fn, args = self._match(self.mapping, web.ctx.path)
    return self._delegate(fn, self.fvars, args)
```

`self._match` 返回的是请求 `request('/a')` 与

```
python
        urls = (
            "/a", "redirect /hello/",
            "/b/(.*)", r"redirect /hello/\1",
            "/hello/(.*)", "hello"
        )
```

进行匹配后的结果， `'/a'` 自然与 `urls[0]`
匹配，得到的 `fn` 是 `"redirect /hello/"`,
得到的 `args` 是 `[]`，因为`urls[0]`里面就没分组。

好了，现在进入 `self._delegate`，看看解析后的
`fn` 和　`args` 被如何处理。

```python
def _delegate(self, f, fvars, args=[]):
    def handle_class(cls):
        meth = web.ctx.method
        if meth == 'HEAD' and not hasattr(cls, meth):
            meth = 'GET'
        if not hasattr(cls, meth):
            raise web.nomethod(cls)
        tocall = getattr(cls(), meth)
        return tocall(\*args)
        
    def is_class(o): return isinstance(o, (types.ClassType, type))
        
    if f is None:
        raise web.notfound()
    elif isinstance(f, application):
        return f.handle_with_processors()
    elif is_class(f):
        return handle_class(f)
    elif isinstance(f, basestring):
        if f.startswith('redirect '):
            url = f.split(' ', 1)[1]
            if web.ctx.method == "GET":
                x = web.ctx.env.get('QUERY_STRING', '')
                if x:
                    url += '?' + x
            raise web.redirect(url)
        elif '.' in f:
            mod, cls = f.rsplit('.', 1)
            mod = __import__(mod, None, None, [''])
            cls = getattr(mod, cls)
        else:
            cls = fvars[f]
        return handle_class(cls)
    elif hasattr(f, '__call__'):
        return f()
    else:
        return web.notfound()
```

一眼就可以看出有一个 `elif` 分支对 `redirect` 进行了处理。

```python
elif isinstance(f, basestring):
    if f.startswith('redirect '):
        url = f.split(' ', 1)[1]
        if web.ctx.method == "GET":
            x = web.ctx.env.get('QUERY_STRING', '')
            if x:
                url += '?' + x
        raise web.redirect(url)
```

分析出 `redirect /hello/` 中的 `/hello/`，再看看
有没有查询字串，就是请求里有没有`?xx` 什么的。
得到最终的 `url` ，转给 `redirect` 函数处理。
直接找到 `webapi.py`，看到 `redirect` 的定义。


```python
class Redirect(HTTPError):
    """A `301 Moved Permanently` redirect."""
    def __init__(self, url, status='301 Moved Permanently', absolute=False):
        """
        Returns a `status` redirect to the new URL. 
        `url` is joined with the base URL so that things like 
        `redirect("about") will work properly.
        """
        newloc = urlparse.urljoin(ctx.path, url)

        if newloc.startswith('/'):
            if absolute:
                home = ctx.realhome
            else:
                home = ctx.home
            newloc = home + newloc

        headers = {
            'Content-Type': 'text/html',
            'Location': newloc
        }
        HTTPError.__init__(self, status, headers, "")

redirect = Redirect
```

这时候传过来的 `url` 是 `/hello/`。 注意 `status`
的默认值。
先使用 `urljoin` 得到一个绝对路径。官网上给出的
`urlparse.urljoin` 的例子是：

```python
>>> from urlparse import urljoin
>>> urljoin('http://www.cwi.nl/%7Eguido/Python.html', 'FAQ.html')
'http://www.cwi.nl/%7Eguido/FAQ.html'
```

另外一个例子是针对第二个参数是绝对路径的情况的。

```python
>>> urljoin('http://www.cwi.nl/%7Eguido/Python.html',
...         '//www.python.org/%7Eguido')
'http://www.python.org/%7Eguido'
```

第二个例子的用法和我们当前的状况一致。所以得到的
`newloc` 还是 `/hello/`。

然后再查看 `ctx.home` 和 `ctx.realhome` 得到
新的　`newloc`。

在 `application.py` 中， `load` 函数定义了一些 `path`:

```python
ctx.homedomain = ctx.protocol + '://' + env.get('HTTP_HOST', '[unknown]')
ctx.homepath = os.environ.get('REAL_SCRIPT_NAME', env.get('SCRIPT_NAME', ''))
ctx.home = ctx.homedomain + ctx.homepath
#@@ home is changed when the request is handled to a sub-application.
#@@ but the real home is required for doing absolute redirects.
ctx.realhome = ctx.home
```

从这可以看出，如果调用子程序　`ctx.home` 会改变，
但是　`ctx.realhome` 不会变，它用来在 redirect 时
生成绝对路径。

```python
if newloc.startswith('/'):
    if absolute:
        home = ctx.realhome
    else:
        home = ctx.home
    newloc = home + newloc
```

现在可以理解了，如果 `absolute` 为真，那就用
`ctx.realhome` 和 `newloc` 组成新值，
如果不为真，直接用 `ctx.home`。这可能在使用子程序，
可以转向到以子程序为基础的`url` 中。

现在回到 `Redirect`。转到 `HTTPError`。

```python
class HTTPError(Exception):
    def __init__(self, status, headers={}, data=""):
        ctx.status = status
        for k, v in headers.items():
            header(k, v)
        self.data = data
        Exception.__init__(self, status)
```

`ctx.status` 被设置，调用 `Exception`。

之后我们回到　`web.redirect(status)` 调用处。

```python
def _delegate(self, f, fvars, args=[]):

    ...

    elif isinstance(f, basestring):
        if f.startswith('redirect '):
            url = f.split(' ', 1)[1]
            if web.ctx.method == "GET":
                x = web.ctx.env.get('QUERY_STRING', '')
                if x:
                    url += '?' + x
            raise web.redirect(url)


    ...

```

可以看出，这里把异常抛出。所以我们回到上一层，看看
相应的处理。

上一层：

```python
def handle(self):
    fn, args = self._match(self.mapping, web.ctx.path)
    return self._delegate(fn, self.fvars, args)
```

再上一层：

```python
# web.application.handle_with_processors

def handle_with_processors(self):
    def process(processors):
        try:
            if processors:
                p, processors = processors[0], processors[1:]
                return p(lambda: process(processors))
            else:
                return self.handle()
        except web.HTTPError:
            raise
        except (KeyboardInterrupt, SystemExit):
            raise
        except:
            print >> web.debug, traceback.format_exc()
            raise self.internalerror()
    
    # processors must be applied in the resvere order. (??)
    return process(self.processors)
```

再上一层：

```python
# web.application.wsgifunc.wsgi

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
```

这里终于看到了对异常的处理。这里 `e.data` 为空。我
们之前并没有设置这个值。

接着看后续处理, `safestr` 在 `utils.py` 中定义，
它负责把给定的对象转化成 `utf-8` 编码的字符串。
下一句设置 `status` 和 `headers` 的值。之前

我们已经看到 `web.ctx.status` 被设置了:

```python
class HTTPError(Exception):
    def __init__(self, status, headers={}, data=""):
        ctx.status = status
        for k, v in headers.items():
            header(k, v)
        self.data = data
        Exception.__init__(self, status)
```

`__init__` 中的 `status` 参数在上一层中设置。

```python
class Redirect(HTTPError):
    """A `301 Moved Permanently` redirect."""
    def __init__(self, url, status='301 Moved Permanently', absolute=False):
        """
        Returns a `status` redirect to the new URL. 
        `url` is joined with the base URL so that things like 
        `redirect("about") will work properly.
        """
        newloc = urlparse.urljoin(ctx.path, url)

        if newloc.startswith('/'):
            if absolute:
                home = ctx.realhome
            else:
                home = ctx.home
            newloc = home + newloc

        headers = {
            'Content-Type': 'text/html',
            'Location': newloc
        }
        HTTPError.__init__(self, status, headers, "")

redirect = Redirect
```

注意这里的　`status` 默认参数。

现在再回到 

```python
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
```

随后调用　`start_resp` ，对 `status`, `headers`
进行处理。我们再回到上一层。

```python
#web.application.request

def request(self, localpart='/', method='GET', data=None,
                host="0.0.0.0:8080", headers=None, https=False, **kw):

        ...

        response = web.storage()
        def start_response(status, headers):
            response.status = status
            response.headers = dict(headers)
            response.header_items = headers
        response.data = "".join(self.wsgifunc()(env, start_response))
        return response
```

这时候，`response` 中的 `stauts`　会设置好。
最后，返回最初的调用。

```python
def testRedirect(self):
    urls = (
        "/a", "redirect /hello/",
        "/b/(.*)", r"redirect /hello/\1",
        "/hello/(.*)", "hello"
    )
    app = web.application(urls, locals())
    class hello:
        def GET(self, name): 
            name = name or 'world'
            return "hello " + name
        
    response = app.request('/a')
    self.assertEquals(response.status, '301 Moved Permanently')
    self.assertEquals(response.headers['Location'], 'http://0.0.0.0:8080/hello/')
```

`self.assertEquals` 会验证 `response.status` 的值。

所以，到这里，`testRedirect` 函数就分析结束了。
