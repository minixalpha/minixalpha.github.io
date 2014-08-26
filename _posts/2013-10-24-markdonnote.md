---
layout: default
title: 使用 Markdown
category: 工具
comments: true
---

# 为什么使用 Markdown

* 看上去不错  
* 既然看上去不错，为什么不试试呢  

---

# 如何使用 Markdonw

## 1. 标题级别

### 效果

# 一级标题  
## 二级标题
### 三级标题

### 代码
```
# 一级标题  
## 二级标题
### 三级标题

```

---

## 2. 列表

### 效果

* 列表1
* 列表2
* 列表3

### 代码 (注意星号与文字之间的空格)

```
* 列表1
* 列表2
* 列表3
```


### 效果

1. 有序列表1
2. 有序列表2
3. 有序列表3


### 代码 (注意点号与文字之间的空格)
```
1. 有序列表1
2. 有序列表2
3. 有序列表3
```

---

## 3. 引用

### 行内代码引用

#### 效果

来试试 `Markdown` 吧

#### 代码

```
来试试 `Markdown` 吧
```


### 代码块引用

#### 效果

```python
def hello():
    print 'hello, world'
```

#### 代码

    ```python
    def hello():
        print 'hello, world'
    ```


### 文字引用

#### 效果

> 春 眠 不 觉 晓，
> 处 处 闻 啼 鸟。  
> 夜 来 风 雨 声，
> 花 落 知 多 少。

#### 代码(需要换行可在句末加两个空格)

```
> 春 眠 不 觉 晓，
> 处 处 闻 啼 鸟。  
> 夜 来 风 雨 声，
> 花 落 知 多 少。
```


#### 效果

> 春 眠 不 觉 晓，
处 处 闻 啼 鸟。  
夜 来 风 雨 声，
花 落 知 多 少。

#### 代码(需要换行可在句末加两个空格)

```
> 春 眠 不 觉 晓，
处 处 闻 啼 鸟。  
夜 来 风 雨 声，
花 落 知 多 少。
```


#### 效果
    春 眠 不 觉 晓，
    处 处 闻 啼 鸟。
    夜 来 风 雨 声，
    花 落 知 多 少。

#### 代码 (行首四个空格, 末尾无需空格)
```
    春 眠 不 觉 晓，
    处 处 闻 啼 鸟。
    夜 来 风 雨 声，
    花 落 知 多 少。
```

---

## 4. 文字

### 斜体

#### 效果

这是一个*斜体*

#### 代码

    这是一个*斜体*

### 粗体

#### 效果

这是一个**粗体**

#### 代码

    这是一个**粗体**

---

## 5. 链接

### 网页链接

#### 效果

参见Wiki词条[Markdown](http://zh.wikipedia.org/wiki/Markdown)

#### 代码

    参见Wiki词条[Markdown](http://zh.wikipedia.org/wiki/Markdown)

### 图片

#### 效果

![Wikipedia_logo](http://upload.wikimedia.org/wikipedia/commons/6/63/Wikipedia-logo.png)

#### 代码

    ![Wikipedia_logo](http://upload.wikimedia.org/wikipedia/commons/6/63/Wikipedia-logo.png)

---

## 6. 公式

### 效果

$$ \sum^{j-1}\_{k=0}{\widehat{\gamma}\_{kj} z_k} $$

### 代码

    $$ \sum^{j-1}\_{k=0}{\widehat{\gamma}\_{kj} z_k} $$


公式可以通过MathJax支持, 右击公式有惊喜。  
另外，注意如果 markdown 的render 是 redcarpet, `LaTex` 中下划线与Markdown中冲突，所以需要做转义处理

如果不想做转义处理，可以尝试更改 markdown 的 render, 我现在用的是 redcarpet,可以改成 kramdown. 如果用的是　jekyll 基于　markdown 搭建博客，可以在　_config.yml 中设置 render，参考　[jekyll configuration](http://jekyllrb.com/docs/configuration/). 感谢评论中 Wead Hsu 提出这一点。


另外，有的　render 在解释文章时可能会出问题，换一个　render 可能就解决了，可以自己实验一下。


特别感谢 [Cmd Markdown](http://www.zybuluo.com/mdeditor?url=http://www.zybuluo.com/static/editor/md-help.markdown)教我使用Markdown
