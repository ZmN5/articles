---
title: "初始化开发环境"
date: 2019-06-30T17:40:01+08:00
lastmod: 2019-06-30T17:40:01+08:00
draft: false
keywords: ["开发环境"]
description: ""
tags: ["开发环境", "工具"]
categories: ["工具"]
author: "何时"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---
最近入职新公司，用自己的电脑没有补贴，那当然用公司电脑喽，所以所有的环境都要重装。通过这次重装确认了几个对我来说，本地开发一定会用到的软件。

1. brew

   1. emmm，这个就不解释了，在mac下时必备的

2. git

   1. brew install git

3. Tmux

   1. 安装： brew install tmux
   2. 安装tpm：https://github.com/tmux-plugins/tpm
   3. 配置参考：https://github.com/fucangyu/my-config/blob/master/dot.tmux.conf

4. Vim8

   1. brew reinstall vim --with-python3 --with-luajit
   2. 安装插件管理工具Plug: https://github.com/junegunn/vim-plug
   3. 安装`universal-tags`:  https://github.com/universal-ctags/ctags
   4. 安装插件： https://github.com/fucangyu/my-config/blob/master/dot.vimrc

5. zsh/oh-my-zsh

   1. 安装`zsh`: https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH
   2. 安装`oh-my-zsh`: https://github.com/robbyrussell/oh-my-zsh
   3. 安装包管理器`zplug`： brew install zplug

6. 安装fd： 

   1. 安装： https://github.com/sharkdp/fd

7. fzf

   1. 安装： brew install fzf
   2. $(brew --prefix)/opt/fzf/install

8. ag

   1. 安装参考： https://github.com/ggreer/the_silver_searcher

9. z jump

   1. 安装参考： https://github.com/rupa/z

10. vscode

    写前端代码时候，感觉用vim还是难受

11. typora

    试过很多个`markdown`编辑工具，感觉还是这个比较好用
<!--more-->
