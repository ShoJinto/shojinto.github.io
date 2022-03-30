---
title: 巧用VSCode实现front-matter自动生成
date: 2022-01-25 23:12:02 +0800
categories: [博客搭建, 技巧]
tags: [博客]
author: 
  name: ShoJinto 
  link: https://shojinto.github.io
---

# 巧用VSCode实现front-matter自动生成

接`博客搭建备忘`Jekyll将`.md`文件转换成静态页面需要依赖`Front-matter`信息来对整个博客进行组织。至于什么是`Front-matter`可以Google查下，这里就不搬运了。

由于每写片文件就需要编辑`Front-matter`，里面的诸如`date`,`author`,`link`等信息都是固定且需要根据实际情况进行动态更新。有没有方法自动生成这些内容呢？一开始就想到有没有类似于`PyCharm`中新建代码文件的时候自定义的模板文件。于是Google了一番还真有，只是不是`Typora`的而是`VSCode`有个`file template`插件，可以实现。同时这个插件还默认内置了：

- JavaScript
- HTML
- CSS
- PHP
- Python
- Ruby
- XML
- Vue

虽然没有内置`markdown`的支持，不过没关系它支持自己定义模板，只需要在`~/.vscode/extensions/ralfzhang.filetemplate-2.0.4/asset/templates`目录下新建一个`.tmpl`文件即可。文件内容仿照目录里面其他文件的内容填写即可。我这边`markdown`的模板文件内容如下：
```yml
---
title: ${1:title}
date: ${date} +0800
categories: [${2:categories}]
tags: [${3:tag}]
author:
  name: ShoJinto
  link: https://shojinto.github.io
---
$0
```
大概解释一下变量的意思：`${1:title}`导入模板之后光标定位的第一个位置，其中`1`表示按`Tab`键的时候光标的切换顺序。

### 设置模板快捷键
我这里设置成`Alt+M`

### 操作步骤：
- 在`VSCode`中`Ctrl+N`新建文件
- `Ctrl+K M`选择文件类型为`markdown`
- `Alt+M`导入模板信息并修改需要修改的内容
- 撰写文章内容

### 致谢

感谢插件的制作者：https://github.com/RalfZhang/File-Template

### 今后markdown的编写工具切换到VSCode上咯！！