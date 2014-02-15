---
layout: default
title: web.py 源代码分析之　web.test.application
category: 源代码阅读
comments: true
---

## 分模块测试

### application.py

对 application.py 的测试，调用命令：

    python test/application.py


程序入口依然会调用 `webtest.main()`

```python
if __name__ == '__main__':
    webtest.main()
```

再仔细观察 `webtest.main` 调用的 `web.test` 中的 `main`函数

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

回顾一下 `module_suite`函数，

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

与 `alltests.py` 不同，`application.py`中并没有 `suite`，
所以直接进入第三分支。后面的流程就和之前讲的后半部分一致了。

这里，我们主要集中于 `application.py`中的具体内容。

* 1. test_reloader

* 2. test_UppercaseMethods

* 3. testRedirect

