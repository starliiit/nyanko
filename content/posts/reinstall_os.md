---
title: "折腾系统配置"
date: 2019-03-09T05:14:43+08:00
tags: ["tools"]
draft: false
---

在家目录下面  `ls -al` 了一下，看到一堆狗屎之后就又想装系统了。

于是我就装了。

## dotfiles 管理

Reference：[A simpler way to manage your dotfiles ](https://www.anand-iyer.com/blog/2018/a-simpler-way-to-manage-your-dotfiles.html)

### 操作

1. 使用一个独立的文件夹 X 保存所有 dotfile。

2. 使用 `git init --bare X` 将 X 建立为一个「裸库」 。

3. 修改 `.zshrc`，使用

  ```
  alias dotfiles='/usr/bin/git --git-dir=$HOME/X/ --work-tree=$HOME'
  ```
  添加一个 `dotfiles` 命令。

4. 之后就可以使用 `dotfiles add, dotfiles commit, dotfiles push` 来对 dotfile 进行管理了。

### 恢复

在新机器上，使用 `git —separate-git-dir` 选项进行 clone:

```
git clone --separate-git-dir=$HOME/.dotfiles https://github.com/starliiit/.dotfiles.git ~
```

如果有程序在 clone 之前创建了同名的配置文件，clone 会失效。可以先 clone 到一个单独的文件夹然后 mv 过去。

### 原理

#### 1. 裸库与普通库的区别

裸库与普通库的区别在于：裸库中只有 `.git` 目录，没有其他源文件（称为 work-tree）。

裸库一般作为远端仓库存在，并且约定俗成使用 `.git` 结尾。我们平常从 github 上进行 `git clone` 时，都是从裸库克隆得到本地仓库。

个人猜测：在 github 上看到一个项目 X 时，看到 X 目录下有很多文件，但实际上在 github 服务端 X 对应的文件夹中应该只有 `.git`，其他的文件都是压缩保存在 `.git` 里面，只是渲染的时候会读取 index 把 work-tree 的结构显示出来。

而 `git clone` 的过程就是先拷贝 `.git` 文件夹，然后在本地把 `.git` 里的 work-tree 解压出来得到实际的源文件。由于本地的文件夹中既有 `.git` 又有 work-tree，因此本地的文件夹变成了一个普通仓库。

在 `clone` 之后，进行修改然后 `commit`，最后进行 `push`。但我们 `push` 的目标并不是本地仓库，而是 `origin` 的某个分支。这里的 `origin` 实际上指向一个裸库。

因此，只有裸库能够接受 `push` 请求，普通仓库是不能接受 `push` 请求的。

Reference:[Git 本地仓库和裸仓库](https://moelove.info/2016/12/04/Git-%E6%9C%AC%E5%9C%B0%E4%BB%93%E5%BA%93%E5%92%8C%E8%A3%B8%E4%BB%93%E5%BA%93/)

#### 2. 在 git 操作时指定 work-tree 和 git-dir

`dotfiles` 命令实际上是包装过的 `git`，在 `git` 执行过程中显式指定 work-tree 和 git-dir。
经过测试，发现 `git add`操作都需要在一个 work-tree 中执行。

这是符合逻辑的：在使用 git 时，我们先对某些文件做修改(`git add`)，然后把修改注册到 `.git` 中。修改必定基于某个主体（可能是单个文件，也可能是多个文件，由于文件系统是树形的，所以这个主体肯定是树形的），这个主体就是 work-tree。

对于普通仓库，git-dir 和 work-tree 是同一个目录。

对于裸库，git-dir 和 work-tree 是分离的。git-dir 是确定的，work-tree 是每次 `git add` 命令执行时，被修改的那个目录。因此在一个 git-dir 中可以添加来自许多不同地方的文件。每次 `git add` 都会把源文件注册到 git-dir 的 `.git` 中。

我们将 `$HOME` 指定为 work-tree，就可以使用 `dotfiles` 命令向裸库执行 `git add, git commit, git push` 了。

## iTerm 和 zsh 美化

本不想用 oh-my-zsh，但试用了 antigen/zprezto 之后觉得太麻烦了，还是一把梭得了。

zsh 主题和 iTerm 配色使用了[honukai-iterm-zsh](https://github.com/starliiit/honukai-iterm-zsh)

然后安装了 [FiraCode](https://github.com/tonsky/FiraCode) 字体。

iTerm 配置导出为了 .iterm2_profile.json 文件。