---
title: Chripy-发布的文章图片路劲问题
date: 2022-01-22 15:23:32 +0800
categories: [博客搭建, 问题修复]
tags: [博客]     # TAG names should always be lowercase
author:
  name: ShoJinto
  link: https://shojinto.github.io

---


# Chripy-发布的文章图片路劲问题

本博客是采用`Jekyll+Chirpy` 搭建完成，其实就是fork自` Chirpy-starter`项目

本地采用`typora`进行文章编写，`markdown`引用的本地图片在`github pages`上会出现路径问题，导致博客上的图片无法正常展示。

为了解决这个问题，经过查阅资料，发现在只需要修改项目目录下的`tools/deploy.sh`就行。经过一番调试最终的部署脚本[见]`tools/deploy.sh`。

这里把思路说一下：

1. 需要在 jekyll 编译之前将md文件中的图片路径修改成编译之后能正常展示图片的路径
2. GitHub pages 部署完成之后，为了让 main 分支上的内容与本地保持一致。还需要对变动进行回退。

修改的两部分代码如下：

```shell
fix_assets_path() {
  # 本地用typora编写的md文章引用本地图片文件，使用相对路径在本地正常，部署到GitHub pages上出现路径问题
  # 此方法解决了这个问题
  POSTS=`ls _posts`
  for post in ${POSTS};
  do
    sed -i 's#../assets#/assets#g' "_posts/"${post}
  done
  
  git config --global user.name "ShoJinto"
  git config --global user.email "shojinto@github.com"

  # commit changes
  git add -A
  git commit -m "fix assets abspath"
  git push -f
}
# 此函数要在build前调用
```



```shell
reset_to_last_manual_submission() {
  # 接`fix_assets_path`函数的注释，未达到远程和本地仓库一致。`github pages` 部署结束后还需要将机器人提交的修改回滚回来
  git config --global user.name "ShoJinto"
  git config --global user.email "shojinto@github.com"
  git checkout main
  git reset --hard HEAD^
  git push -f
}
# 此函数要在deploy完之后调用
```



至此大功告成！！！