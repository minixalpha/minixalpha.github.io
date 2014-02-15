---
layout: default
title: web.py 源代码分析之 web.test 主要文件及测试流程
category: 源代码阅读
comments: true
---

# 目录文件说明

## README

如何运行测试文件，包含全部测试及分模块测试

调用

    $ python test/alltests.py

运行全部测试

调用

    $ python test/db.py

运行db模块测试

## alltest.py

运行全部测试入口，调用 `webtest` 模块完成测试

```python
# alltest.py

if __name__ == "__main__":
    webtest.main()
```

## webtest.py

我们发现 webtest.py 中并没有 main 函数，而是从 `web.test` 中导入，

```python
# webtest.py
from web.test import *
```

也就是说，如果 `web.test`中有main函数的话，`webtest.main()`
其实是调用 `web.test` 中的main函数。

感觉～ 好神奇

## web.test

看web目录下的test.py文件，果然发现了main函数，终于找到入口啦~

```python
def main(suite=None):
    if not suite:
        main_module = __import__('__main__')
        # allow command line switches
        args = [a for a in sys.argv[1:] if not a.startswith('-')]
        suite = module_suite(main_module, args or None)

    result = runTests(suite)
    sys.exit(not result.wasSuccessful())
```

把这个main函数改掉，再运行一下：

    $ python test/alltests.py

果然是运行修改后的函数，所以这里确定是入口。

在进入下一步之前，我们需要学习一下Python自动单元测试框架，即`unittest`模块。关于 `unittest` ，可以参考这篇文章：
[Python自动单元测试框架](http://www.ibm.com/developerworks/cn/linux/l-pyunit/)


现在，进入 `web.test.main`，
当我们如果我们打印 `main_module`，会得到

    <module '__main__' from 'test/alltests.py'>

这说明调用时，`import` 得到的 `main` 与命令行调用指定的
调用的模块一致。

之后会调用 `module_suite` 得到需要测试的模块,

我们看一下 `module_suite` 的代码，

```python
def module_suite(module, classnames=None):
    """Makes a suite from a module."""
    if classnames:
        return unittest.TestLoader().loadTestsFromNames(classnames, module)
    elif hasattr(module, 'suite'):
        return module.suite()
    else:
        return unittest.TestLoader().loadTestsFromModule(module)
```

可以看出，module_suite分三部分，如果定义了 `classnames`，
会测试具体的类，否则，如果 `module` 中含有 `suite` 函数，
就返回此 `module.suite()` 的调用结果。

此时我们的 `module` 是之前得到的 

    <module '__main__' from 'test/alltests.py'>

而 `alltests.py` 中刚好就有 `suite` 函数：

```python
def suite():
    modules = ["doctests", "db", "application", "session"]
    return webtest.suite(modules)
```

`modules` 是全部模块的列表，随后以此为参数，返回
调用 `webtest.suite` 的结果。

这时候，同样的情况又出现了，`webtest.py` 中没有 `suite` 函数，
但是 `webtest.py` 中含有

```python
from web.test import *
```

所以 `webtest.suite` 调用的还是 `web.test.suite`，

从这里我们可以看出，

> webtest.py 这个模块就是目录 `test` 下的模块与 `web.test` 模块
> 之间的一个过渡层，`test` 目录下的模块调用 `webtest.XXX`，而实际
> 的实现代码都是调用 `web.test.XXX`，所以，我们再次回到 `web.test`。

看看 `web.test.suite`的实现：

```python
def suite(module_names):
    """Creates a suite from multiple modules."""
    suite = TestSuite()
    for mod in load_modules(module_names):
        suite.addTest(module_suite(mod))
    return suite
```

首先调用 `TestSuite()`，即 `unittest.TestSuite()`，得到一个
`suite`。然后，向 `suite` 中添加测试用例。

参数 `module_names` 就是刚刚的列表modules：

    modules = ["doctests", "db", "application", "session"]

调用 `load_modules` 把模块名变为模块本身，
例如 "application":

    >>> __import__("application", None, None, "x")
    <module 'application' from 'application.pyc'>

然后，再调用 `module_suite`，这时候你会发现，我们递归回来了，

```python
def module_suite(module, classnames=None):
    """Makes a suite from a module."""
    if classnames:
        return unittest.TestLoader().loadTestsFromNames(classnames, module)
    elif hasattr(module, 'suite'):
        return module.suite()
    else:
        return unittest.TestLoader().loadTestsFromModule(module)
```

此时, `module`为:

    <module 'application' from 'application.pyc'>

我们再看看这时候，`application.py`中是否有 `suite`函数。
经过查看，发现没有，所以此时就会调用 `if`语句第三个分支：

```python
        return unittest.TestLoader().loadTestsFromModule(module)
```

[TestLoader](http://docs.python.org/2/library/unittest.html#unittest.TestLoader) 中的 `loadTestsFromModule` 从模块中导入
测试用例。 这个函数会搜索模块中 `TestCase`类的子类，再创建一个
子类的实例，以便调用子类中的测试函数。
例如 `test/application.py`中的 `ApplicationTest` 类：

```python
class ApplicationTest(webtest.TestCase):
    def test_reloader(self):
        write('foo.py', data % dict(classname='a', output='a'))
        import foo
        app = foo.app
        
        self.assertEquals(app.request('/').data, 'a')
        
        # test class change
        time.sleep(1)
        write('foo.py', data % dict(classname='a', output='b'))
        self.assertEquals(app.request('/').data, 'b')

        # test urls change
        time.sleep(1)
        write('foo.py', data % dict(classname='c', output='c'))
        self.assertEquals(app.request('/').data, 'c')
        
    def testUppercaseMethods(self):
        urls = ("/", "hello")
        app = web.application(urls, locals())
        class hello:
            def GET(self): return "hello"
            def internal(self): return "secret"
            
        response = app.request('/', method='internal')
        self.assertEquals(response.status, '405 Method Not Allowed')

     ...
```

返回的这些函数会经过 `web.test.suite` 中的 `suite.addTest` 加入到
`suite`中。 这样经过循环，所有模块中的测试用例就加入 `suite`中。

之后，我们回到 `web.test.main` 中，调用

```python
result = runTests(suite)
```

完成最终的测试。 `runTests` 在 `web.test`模块中：

```python
def runTests(suite):
    runner = unittest.TextTestRunner()
    return runner.run(suite)
```

`runner`功能同样由 `unittest`模块提供。

到这里，我们应该明白了测试的整个过程的全部细节。

这里，我们对 `alltests.py` 完成的功能做一个总结。
从总体上看，`alltests.py`对所有模块完成测试,包括：

```
    modules = ["doctests", "db", "application", "session"]
```

我们绕来绕去，其实不过下列过程：

* 通过 `suite=TestSuite()` 得到 `suite`
* 把所有模块中的 test 函数  通过 `suite.addTest()` 加入 `suite`
* 通过 `runner=unittest.TextTestRunner()` 得到 `runner`
* 运行 `runner.run(suite)` 调用所有测试用例。

之所以感觉绕，是因为 层次关系，及`alltests.py` 需要递归调用 `module_suite`:

* `alltests.py` 调用 `webtest.main`
* `webtest.main` 指向 `web.test.main` 
* 由`web.test.main` 进入 `web.test.module_suite`，进入 `if` 第二分支
* 再进入 `alltest.py`，调用 `suite()`
* 再进入 `web.test.suite` , 对每个模块调用 `web.test.module_suite`
* 再进入 `web.test.module_suite`，进入 `if` 第三分支,得到每个模块中的 测试用例
* 将所有测试用例加入 `suite()` 中的 `suite`


## requirements.txt

requirements.txt 文件可由 pip 生成：

    $pip freeze -l > requirements.txt

同时，pip 可以使用 requirements.txt 文件安装依赖包

    $pip install -r requirements.txt

这就为打包与安装包提供了方便

