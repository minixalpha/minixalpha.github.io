---
layout: default
title: WSGI 简介
category: 技术
comments: true
---

#WSGI 简介 

## 背景
Python Web 开发中，服务端程序可以分为两个部分，一是服务器程序，二是应用程序。前者负责把客户端请求接收，整理，后者负责具体的逻辑处理。为了方便应用程序的开发，我们把常用的功能封装起来，成为各种Web开发框架，例如 Django, Flask, Tornado。不同的框架有不同的开发方式，但是无论如何，开发出的应用程序都要和服务器程序配合，才能为用户提供服务。这样，服务器程序就需要为不同的框架提供不同的支持。这样混乱的局面无论对于服务器还是框架，都是不好的。对服务器来说，需要支持各种不同框架，对框架来说，只有支持它的服务器才能被开发出的应用使用。

这时候，标准化就变得尤为重要。我们可以设立一个标准，只要服务器程序支持这个标准，框架也支持这个标准，那么他们就可以配合使用。一旦标准确定，双方各自实现。这样，服务器可以支持更多支持标准的框架，框架也可以使用更多支持标准的服务器。

Python Web开发中，这个标准就是 *The Web Server Gateway Interface*, 即 **WSGI**. 这个标准在[PEP 333](http://www.python.org/dev/peps/pep-0333/)中描述，后来，为了支持 Python 3.x, 并且修正一些问题，新的版本在[PEP 3333](http://www.python.org/dev/peps/pep-3333/)中描述。

## WSGI 是什么 
WSGI 是服务器程序与应用程序的一个约定，它规定了双方各自需要实现什么接口，提供什么功能，以便二者能够配合使用。

WSGI 不能规定的太复杂，否则对已有的服务器来说，实现起来会困难，不利于WSGI的普及。同时WSGI也不能规定的太多，例如cookie处理就没有在WSGI中规定，这是为了给框架最大的灵活性。要知道WSGI最终的目的是为了方便服务器与应用程序配合使用，而不是成为一个Web框架的标准。

另一方面，WSGI需要使得middleware（是中间件么？）易于实现。middleware处于服务器程序与应用程序之间，对服务器程序来说，它相当于应用程序，对应用程序来说，它相当于服务器程序。这样，对用户请求的处理，可以变成多个 middleware 叠加在一起，每个middleware实现不同的功能。请求从服务器来的时候，依次通过middleware，响应从应用程序返回的时候，反向通过层层middleware。我们可以方便地添加，替换middleware，以便对用户请求作出不同的处理。


## WSGI 内容概要
WSGI主要是对应用程序与服务器端的一些规定，所以，它的主要内容就分为两个部分。

## 应用程序

WSGI规定：

    1. 应用程序需要是一个可调用的对象

在Python中:

* 可以是函数
* 可以是一个实例，它的类实现了`__call__`方法
* 可以是一个类，这时候，用这个类生成实例的过程就相当于调用这个类

同时，WSGI规定：

    2. 可调用对象接收两个参数

这样，如果这个对象是函数的话，它看起来要是这个样子：

```python
# callable function
def application(environ, start_response):
    pass
```

如果这个对象是一个类的话，它看起来是这个样子：

```python
# callable class
class Application:
    def __init__(self, environ, start_response):
        pass
```

如果这个对象是一个类的实例，那么，这个类看起来是这个样子：

```python
# callable object
class ApplicationObj:
    def __call__(self, environ, start_response):
        pass
```

最后，WSGI还规定:
    
    3.可调用对象要返回一个值，这个值是可迭代的。

这样的话，前面的三个例子就变成：

```python
HELLO_WORLD = b"Hello world!\n"


# callable function
def application(environ, start_response):
    return [HELLO_WORLD]


# callable class
class Application:
    def __init__(self, environ, start_response):
        pass

    def __iter__(self):
        yield HELLO_WORLD


# callable object
class ApplicationObj:
    def __call__(self, environ, start_response):
        return [HELLO_WORLD]
```

你可能会说，不是啊，我们平时写的web程序不是这样啊。
比如如果使用web.py框架的话，一个典型的应用可能是这样的:

```python
class hello:
    def GET(self):
        return 'Hello, world!'
```

这是由于框架已经把WSGI中规定的一些东西封装起来了，我们平时用框架时，看不到这些东西，只需要直接实现我们的逻辑，再返回一个值就好了。其它的东西框架帮我们做好了。这也是框架的价值所在，把常用的东西封装起来，让使用者只需要关注最重要的东西。

当然，**WSGI关于应用程序的规定不只这些**，但是现在，我们只需要知道这些就足够了。下面，再介绍服务器程序。

## 服务器程序

服务器程序会在每次客户端的请求传来时，调用我们写好的应用程序，并将处理好的结果返回给客户端。

WSGI规定：

    4.服务器程序需要调用应用程序

服务器程序看起来大概是这个样子的：

```python
def run(application):
    environ = {}

    def start_response(status, response_headers, exc_info=None):
        pass

    result = application(environ, start_response)

    def write(data):
        pass

    for data in result:
        write(data)
```

这里可以看出服务器程序是如何与应用程序配合完成用户请求的。

WSGI规定了应用程序需要一个可调用对象，有两个参数，返回一个可迭代对象。在服务器
程序中，针对这几个规定，做了以下几件事：

* 把应用程序需要的两个参数设置好
* 调用应用程序
* 迭代访问应用程序的返回结果，并将其传回客户端

你可以从中发现，应用程序需要的两个参数，一个是一个dict对象，一个是函数。它们到底有什么用呢？这都不是我们现在应该关心的，现在只需要知道，服务器程序大概做了什么事情就好了，后面，我们会深入讨论这些细节。

## middleware

另外，有些功能可能介于服务器程序和应用程序之间，例如，服务器拿到了客户端请求的URL,
不同的URL需要交由不同的函数处理，这个功能叫做 URL Routing，这个功能就可以放在二者中间实现，这个中间层就是 middleware。

middleware对服务器程序和应用是透明的，也就是说，服务器程序以为它就是应用程序，而应用程序以为它就是服务器。这就告诉我们，middleware需要把自己伪装成一个服务器，接受应用程序，调用它，同时middleware还需要把自己伪装成一个应用程序，传给服务器程序。

其实无论是服务器程序，middleware 还是应用程序，都在服务端，为客户端提供服务，之所以把他们抽象成不同层，就是为了控制复杂度，使得每一次都不太复杂，各司其职。

下面，我们看看middleware大概是什么样子的。

```python
# URL Routing middleware
def urlrouting(url_app_mapping):
    def midware_app(environ, start_response):
        url = environ['PATH_INFO']
        app = url_app_mapping[url]

        result = app(environ, start_response)

        return result

    return midware_app
```

函数 `midware_app`就是一个简单的middleware：对服务器而言，它是一个应用程序，是一个可调用对象， 有两个参数，返回一个可调用对象。对应用程序而言，它是一个服务器，为应用程序提供了参数，并且调用了应用程序。

另外，这里的`urlrouting`函数，相当于一个函数生成器，你给它不同的 url-app 映射关系，它会生成相应的具有 url routing功能的 middleware。


如果你仅仅想简单了解一下WSGI是什么，相信到这里，你差不多明白了，下面会介绍WSGI的细节，这些细节来自 [PEP3333](http://www.python.org/dev/peps/pep-3333/)， 如果没有兴趣，到这里 **可以停止**了。

## WSGI详解

注意：以 点 开始的解释是WSGI规定 **必须满足** 的。

### 应用程序

* 应用程序是可调用对象
* 可调用对象有两个位置参数  
所谓位置参数就是调用的时候，依靠位置来确定参数的语义，而不是参数名，也就是说服务
器调用应用程序时，应该是这样：

```python
application(env, start_response)
```

而不是这样：

```python
application(start_response=start_response, environ=env)
```

所以，参数名其实是可以随便起的，只不过为了表义清楚，我们起了`environ` 和 `start_response`。

* 第一个参数environ是Python内置的dict对象，应用程序可以对这个参数任意修改。
* environ参数必须包含 WSGI 需要的一些变量(详见后文)  
也可以包含一些扩展参数，命名规范见后文
* start_response参数是一个可调用对象。接受两个位置参数，一个可选参数。
例如：
```python
start_response(status, response_headers, exc_info=None)
```
status参数是状态码，例如 `200 OK` 。 

response_headers参数是一个列表，列表项的形式为(header_name, header_value)。  

exc_info参数在错误处理的时候使用。

status和response_headers的具体内容可以参考 [HTTP 协议 Response部分](http://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html#sec6)。

* start_response必须返回一个可调用对象： `write(body_data)`
* 应用程序必须返回一个可迭代对象。
* 应用程序不应假设返回的可迭代对象被遍历至终止，因为遍历过程可能出现错误。
* 应用程序必须在第一次返回可迭代数据之前调用 start_response 方法。  
这是因为可迭代数据是 返回数据的 `body` 部分，在它返回之前，需要使用 `start_response`
返回 response_headers 数据。

### 服务器程序

* 服务器必须将可迭代对象的内容传递给客户端，可迭代对象会产生bytestrings，必须完全完成每个bytestring后才能请求下一个。
* 假设result 为应用程序的返回的可迭代对象。如果len(result) 调用成功，那么result必须是可累积的。
* 如果result有`close`方法，那么每次完成对请求的处理时，必须调用它，无论这次请求正常完成，还是遇到了错误。
* 服务器程序禁止使用可迭代对象的其它属性，除非这个可迭代对象是一个特殊类的实例，这个类会被 `wsgi.file_wrapper` 定义。


根据上述内容，我们的服务器程序看起来会是这个样子：

```python
def run(application):
    environ = {}

    # set environ
    def write(data):
        pass

    def start_response(status, response_headers, exc_info=None):
        return write

    try:
        result = application(environ, start_response)
    finally:
        if hasattr(result, 'close'):
            result.close()

    if hasattr(result, '__len__'):
        # result must be accumulated
        pass


    for data in result:
        write(data)
```

应用程序看起来是这个样子：

```python
HELLO_WORLD = b"Hello world!\n"


# callable function
def application(environ, start_response):
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)

    return [HELLO_WORLD]

```


下面我们再详细介绍之前提到的一些数据结构

### environ 变量 

environ 变量需要包含 CGI 环境变量，它们在[The Common Gateway Interface Specification](http://tools.ietf.org/html/draft-robinson-www-interface-00) 中定义，下面列出的变量**必须**包含在 enciron变量中：

* REQUEST_METHOD  
HTTP 请求方法，例如 "GET", "POST"
* SCRIPT_NAME  
URL 路径的起始部分对应的应用程序对象，如果应用程序对象对应服务器的根，那么这个值可以为空字符串
* PATH_INFO  
URL 路径除了起始部分后的剩余部分，用于找到相应的应用程序对象，如果请求的路径就是根路径，这个值为空字符串
* QUERY_STRING  
URL路径中 `?` 后面的部分
* CONTENT_TYPE  
HTTP 请求中的 `Content-Type` 部分
* CONTENT_LENGTH  
HTTP 请求中的`Content-Lengh` 部分
* SERVER_NAME, SERVER_PORT  
与 SCRIPT_NAME，PATH_INFO 共同构成完整的  URL，它们永远不会为空。但是，如果 HTTP_HOST 存在的话，当构建 URL 时， HTTP_HOST优先于SERVER_NAME。
* SERVER_PROTOCOL  
客户端使用的协议，例如 "HTTP/1.0", "HTTP/1.1", 它决定了如何处理 HTTP 请求的头部。这个名字其实应该叫 `REQUEST_PROTOCOL`，因为它表示的是客户端请求的协议，而不是服务端响应的协议。但是为了和CGI兼容，我们只好叫这个名字了。
*HTTP_ Variables  
这个是一个系列的变量名，都以`HTTP`开头，对应客户端支持的HTTP请求的头部信息。

WSGI 有一个参考实现，叫 wsgiref，里面有一个示例，我们这里引用这个示例的结果，展现一下这些变量，以便有一个直观的体会，这个示例访问的 URL 为 `http://localhost:8000/xyz?abc`

上面提到的变量值为：

```python
REQUEST_METHOD = 'GET'
SCRIPT_NAME = ''
PATH_INFO = '/xyz'
QUERY_STRING = 'abc'
CONTENT_TYPE = 'text/plain'
CONTENT_LENGTH = ''
SERVER_NAME = 'minix-ubuntu-desktop'
SERVER_PORT = '8000'
SERVER_PROTOCOL = 'HTTP/1.1'

HTTP_ACCEPT = 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'
HTTP_ACCEPT_ENCODING = 'gzip,deflate,sdch'
HTTP_ACCEPT_LANGUAGE = 'en-US,en;q=0.8,zh;q=0.6,zh-CN;q=0.4,zh-TW;q=0.2'
HTTP_CONNECTION = 'keep-alive'
HTTP_HOST = 'localhost:8000'
HTTP_USER_AGENT = 'Mozilla/5.0 (X11; Linux i686) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36'
```

另外，服务器还应该（非必须）提供尽可能多的CGI变量，如果支持SSL的话，还应该提供[Apache SSL 环境变量](http://www.modssl.org/docs/2.8/ssl_reference.html#ToC25)。

服务器程序应该在文档中对它提供的变量进行说明，应用程序应该检查它需要的变量是否存在。

除了 CGI 定义的变量外，服务器程序还可以包含和操作系统相关的环境变量，但这并非必须。

但是，下面列出的这些 WSGI 相关的变量必须要包含：

* wsgi.version  
值的形式为 (1, 0) 表示 WSGI 版本 1.0
* wsgi.url_scheme  
表示 url 的模式，例如 "https" 还是 "http"
* wsgi.input  
输入流，HTTP请求的 body  部分可以从这里读取
* wsgi.erros  
输出流，如果出现错误，可以写往这里
* wsgi.multithread  
如果应用程序对象可以被同一进程中的另一线程同时调用，这个值为True
* wsgi.multiprocess  
如果应用程序对象可以同时被另一个进程调用，这个值为True
* wsgi.run_once  
如果服务器希望应用程序对象在包含它的进程中只被调用一次，那么这个值为True

这些值在 wsgiref示例中的值为：

```python
wsgi.errors = <open file '<stderr>', mode 'w' at 0xb735f0d0>
wsgi.file_wrapper = <class wsgiref.util.FileWrapper at 0xb70525fc>
wsgi.input = <socket._fileobject object at 0xb7050e6c>
wsgi.multiprocess = False
wsgi.multithread = True
wsgi.run_once = False
wsgi.url_scheme = 'http'
wsgi.version = (1, 0)
```

另外，environ中还可以包含服务器自己定义的一些变量，这些变量应该只包含  
* 小写字母
* 数字
* 点
* 下划线
* 独立的前缀

例如，mod_python定义的变量名应该为mod_python.var_name的形式。

## 输入流及错误流(Input and Error Streams)

服务器程序提供的输入流及错误流必须包含以下方法：

* read(size)
* readline()
* readlines(hint)
* __iter__()
* flush()
* write()
* writelines(seq)

应用程序使用输入流对象及错误流对象时，只能使用这些方法，禁止使用其它方法，特别是，
禁止应用程序关闭这些流。

## start_response() 
start_response是HTTP响应的开始，它的形式为：

```python
start_response(status, response_headers, exc_info=None)
```

返回一个可调用对象，这个可调用对象形式为：

```python
write(body_data)
```

status 表示  HTTP 状态码，例如 "200 OK", "404 Not Found"，它们在 [RFC 2616](http://www.faqs.org/rfcs/rfc2616.html)中定义，status禁止包含控制字符。

response_headers 是一个列表，列表项是一个二元组： (header_name, heaer_value) ，
每个 header_name 都必须是 [RFC 2616](http://www.faqs.org/rfcs/rfc2616.html)  4.2 节中定义的HTTP 头部名。header_value 禁止包含控制字符。

另外，服务器程序必须保证正确的headers 被返回给客户端，如果应用程序没有返回headers，服务器必须添加它。

应用程序和middleware禁止使用 HTTP/1.1 中的 "hop-by-hop"特性，以及其它可能影响客户端与服务器永久连接的特性。

start_response 被调用时，服务器应该检查 headers 中的错误，另外，禁止 start_response直接将 response_headers传递给客户端，它必须把它们存储起来，一直到应用程序第一次迭代返回一个非空数据后，才能将response_headers传递给客户端。这其实是在说，HTTP响应body部分必须有数据，不能只返回一个header。

start_response的第三个参数是一个可选参数，exc_info，它必须和Python的 sys.exc_info()返回的数据有相同类型。当处理请求的过程遇到错误时，这个参数会被设置，同时调用 start_response。如果提供了exc_info，但是HTTP headers 还没有输出，那么 start_response需要将当前存储的 HTTP response headers替换成一个新值。但是，如果提供了exc_info，同时 HTTP headers已经输出了，那么 start_response 必须 raise 一个 error。禁止应用程序处理
start_response raise出的  exceptions，应该交给服务器程序处理。

当且仅当提供 exc_info参数时，start_response才可以被调用多于一次。换句话说，要是没提供这个参数，start_response在当前应用程序中调用后，禁止再调用。

为了避免循环引用，start_response实现时需要保证 exc_info在函数调用后不再包含引用。
也就是说start_response用完 exc_info后，需要保证执行一句

```python
exc_info = None
```

这可以通过 try/finally实现。

## 处理 Content-Length Header

如果应用程序支持 Content-Length，那么服务器程序传递的数据大小不应该超过 Content-Length，当发送了足够的数据后，应该停止迭代，或者 raise 一个 error。当然，如果应用程序返回的数据大小没有它指定的Content-Length那么多，那么服务器程序应该关闭连接，使用Log记录，或者报告错误。

如果应用程序不支持Content-Length,那么服务器程序应该选择一种方法处理这种情况。最简单的方法就是当响应完成后，关闭与客户端的连接。

### 缓冲与流(Buffering and Streaming)

一般情况下，应用程序会把需要返回的数据放在缓冲区里，然后一次性发送出去。之前说的应用程序会返回一个可迭代对象，多数情况下，这个可迭代对象，都只有一个元素，这个元素包含了HTML内容。但是在有些情况下，数据太大了，无法一次性在内存中存储这些数据，所以就需要做成一个可迭代对象，每次迭代只发送一块数据。

禁止服务器程序延迟任何一块数据的传送，要么把一块数据完全传递给客户端，要么保证在产生下一块数据时，继续传递这一块数据。

## middleware 处理数据
如果 middleware调用的应用程序产生了数据，那么middleware至少要产生一个数据，即使它想等数据积累到一定程度再返回，它也需要产生一个空的bytestring。
注意，这也意味着只要middleware调用的应用程序产生了一个可迭代对象，middleware也必须返回一个可迭代对象。
同时，禁止middleware使用可调用对象write传递数据，write是middleware调用的应用程序使用的。

### write 可调用对象

一些已经存在的应用程序框架使用了write函数或方法传递数据，并且没有使用缓冲区。不幸的是，根据WSGI中的要求，应用程序需要返回可迭代对象，这样就无法实现这些API,为了允许这些API 继续使用，WSGI要求 start_response 返回一个 write 可调用对象，这样应用程序就能使用这个  write  了。

但是，如果能避免使用这个 write，最好避免使用，这是为兼容以前的应用程序而设计的。这个write的参数是HTTP response body的一部分，这意味着在write()返回前，必须保证传给它的数据已经完全被传送了，或者已经放在缓冲区了。

应用程序必须返回一个可迭代对象，即使它使用write产生HTTP response body。

这里可以发现，有两中传递数据的方式，一种是直接使用write传递，一种是应用程序返回可迭代对象后，再将这个可迭代对象传递，如果同时使用这两种方式，前者的数据必须在后者之前传递。

### Unicode 

HTTP 不支持 Unicode, 所有编码/解码都必须由应用程序完成，所有传递给或者来自server的字符串都必须是 `str` 或者 `bytes`类型，而不是`unicode`。

注意传递给start_response的数据，其编码都必须遵循 [RFC 2616](http://www.faqs.org/rfcs/rfc2616.html)， 即使用 ISO-8859-1  或者  [RFC 2047](http://www.faqs.org/rfcs/rfc2047.html) MIME 编码。

WSGI 中据说的 `bytestrings` ， 在Python3中指 `bytes`，在以前的Python版本中，指
`str`。

### 错误处理(Error Handling)

应用程序应该捕获它们自己的错误，internal erros， 并且将相关错误信息返回给浏览器。
WSGI 提供了一种错误处理的方式，这就是之前提到的 exc_info参数。下面是 PEP 3333中提供的一段示例：

```python
try:
    # regular application code here
    status = "200 Froody"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers)
    return ["normal body goes here"]
except:
    # XXX should trap runtime issues like MemoryError, KeyboardInterrupt
    #     in a separate handler before this bare 'except:'...
    status = "500 Oops"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers, sys.exc_info())
    return ["error body goes here"]
```

当出现异常时，start_response的exc_info参数被设置成 sys.exc_info()，这个函数会返回当前的异常。


### HTTP 1.1 Expect/Continue

如果服务器程序要实现 HTTP 1.1，那么它必须提供对 HTTP 1.1 `expect/continue`机制的支持。

## 其它内容

在 PEP 3333 中，还包含了其它内容，例如：

* HTTP 特性
* 线程支持
* 实现时需要注意的地方：包括，扩展API，应用程序配置，URL重建等 

这里就不作过多介绍了。

## 扩展阅读
这篇文章主要是我阅读 PEP 3333 后的理解和记录，有些地方可能没有理解正确或者没有写全，下面提供一些资源供扩展阅读。

* [PEP 3333](http://www.python.org/dev/peps/pep-3333/)   
不解释
* [WSGI org](http://wsgi.readthedocs.org/en/latest/)  
看起来好像官方网站的样子，覆盖了关于WSGI的方方面面，包含学习资源，支持WSGI的框架列表，服务器列表，应用程序列表，middleware和库等等。
* [wsgiref](https://pypi.python.org/pypi/wsgiref)  
WSGI的参考实现，阅读源代码后有利于对WSGI的理解。我在GitHub上有自己阅读后的注释版本，并且作了一些图，有需要可以看这里：[wsgiref 源代码阅读](https://github.com/minixalpha/SourceLearning/tree/master/wsgiref-0.1.2)

另外，还有一些文章介绍了一些基本概念和一些有用的实例，非常不错。

* [Wsgi研究](http://blog.kenshinx.me/blog/wsgi-research/)
* [wsgi初探](http://linluxiang.iteye.com/blog/799163) 
