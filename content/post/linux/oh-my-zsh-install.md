---
title: "oh-my-zsh Mac 安装教程"
description: "本文针对的mac用户"
tags: ["oh-my-zsh"]
categories: ["Linux"]
date: 2021-02-26T17:14:36+08:00
draft: false
---

这是针对Mac的教程, Windows用户慎用喔

<!--more-->

## 安装oh-my-zsh

```shell
# 安装 git 和 zsh 选择curl或者wget其一命令执行
yum install -y git zsh

# curl
sh -c "$(curl -fsSL https://gitee.com/Devkings/oh_my_zsh_install/raw/master/install.sh)"
# wget
sh -c "$(wget https://gitee.com/Devkings/oh_my_zsh_install/raw/master/install.sh -O -)"
```

## 修改配置文件

```shell
cd && cd .oh-my-zsh/plugins
# 高亮插件
git clone https://gitee.com/sliverhorn/zsh-syntax-highlighting.git

# 记录历史插件
git clone https://gitee.com/sliverhorn/zsh-autosuggestions.git

vi ~/.zshrc
# agnoster是内置主题，所以不需要安装
ZSH_THEME="agnoster"
# 找到对应位置加上zsh-syntax-highlighting zsh-autosuggestions 
plugins=(zsh-syntax-highlighting zsh-autosuggestions git)
# 下面两句是没有相同配置项的，
# 第一个是隐藏用户(有些主题会隐藏，agnoster内置主题不会隐藏)，
# 第二是gf可能冲突，官方推荐这样做的
DEFAULT_USER="sliverhorn"
# alias gf=gf

# 使.zshrc配置文件生效
source ~/.zshrc
```



