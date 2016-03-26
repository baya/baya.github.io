---
layout: post
title: 我的 git cookbook
---

## 代码托管

- https://bitbucket.org/
- github

## alias

- https://git.wiki.kernel.org/index.php/Aliases#Aliases
- 在~/.gitconfig加入,

~~~bash

[alias]
    st = status
    ci = commit
    br = branch
    co = checkout
    df = diff
    dc = diff --cached
    lg = log -p
    who = shortlog -s --
	
~~~	

## 基础教程

- http://rogerdudler.github.io/git-guide/index.zh.html
- http://git-scm.com/book/zh/

### 彩色的git输出

`git config color.ui true`

## 建立git服务

`git init --bare`

## git log

`git log --reverse   shows commits from start`

## code review

~~~bash
$ git log --until=2013-09-12                    # 假设今天是2013-09-13，此命令可以找出昨天最后一次的commit号，假设是abcdef
$ git diff abcdef                               # 此命令就可以查看今天一整天所做的修改
$ git diff abcdef > code-reveiw-20130913.diff   # 将diff后的结果输入到code-review-20130913.diff文件，方便review
~~~
### 写成了一个rake task

~~~ruby
namespace :git do
  task :diff do
    tt = Time.now
    yt = Time.now - (60 * 60 * 24)
    ys = "\"#{yt.strftime('%F')} 23:59:59\""
    ts = tt.strftime('%Y%m%d')
    log = `git log --until=#{ys} -1`
    commit = log.split("\n")[0].split(' ')[1]
    diff = `git diff #{commit}`
    puts diff
    File.open("code_review_#{ts}.diff", 'w+') {|f| f.write diff }
  end
end

~~~

执行 `rake git:diff` 就能对今天的代码进行 review 了

## 简明reference

- http://gitref.org/

## 推荐的.gitignore

~~~bash
# Compiled source #
###################
*.com
*.class
*.dll
*.exe
*.o
*.so

# Packages #
############
# it's better to unpack these files and commit the raw source
# git has its own built in compression methods
*.7z
*.dmg
*.gz
*.iso
*.jar
*.rar
*.tar
*.zip

# Logs and databases #
######################
*.log
*.sql
*.sqlite

# OS generated files #
######################
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

~~~

## git init --bare 创建一个裸仓库
- 参考: http://git-scm.com/book/zh/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E9%83%A8%E7%BD%B2-Git

~~~bash

mkdir fchk-api-doc.git
cd fchk-api-doc.git
git init --bare --shared

# 推送代码到远程裸仓库
git remote add fchk-staging ssh://staging.fchk.com/home/jgm/git/fchk-api-doc.git/
git push fchk-staging master

## setting username
$ git config --global user.name "Billy Everyteen"
# Set a new name
$ git config --global user.name
# Verify the setting
# Billy Everyteen

~~~

## utf-8支持

- http://stackoverflow.com/questions/5854967/git-msysgit-accents-utf-8-the-definitive-answers

~~~bash
git config [--global] core.quotepath off
git config [--global] i18n.logoutputencoding utf8
git config [--global] i18n.commitencoding utf8
git config [--global] --unset svn.pathnameencoding
~~~

## 回滚到某个commit

`git reset --hard commit`

实例:

`git reset --hard b0ae0c3e5e0430853430fc71a4ec3756abe2b2b9`

## 查看某个文件的历史变化

- http://stackoverflow.com/questions/278192/view-the-change-history-of-a-file-using-git-versioning

~~~bash
$ gitk [filename]
$ git log --follow -p file
~~~

## 修复未提交文件中的错误(重置)

- http://gitbook.liuhui998.com/4_9.html
在工作目录做了一些糟糕的修改，但是还没有提交，想恢复到上次提交时的状态

~~~bash
$ git reset --hard HEAD
~~~

## 签出特定的commit

- http://stackoverflow.com/questions/2007662/rollback-to-an-old-commit-using-git
我发现rails_admin的交互突然不能用了，没有发现js错误，所以我想知道是从那个commit开始出现这种问题的。

~~~bash
$ git checkout [revision] .
~~~

调试过历史版本后，想恢复到最近提交的版本，使用:

~~~bash
$ git reset --hard HEAD
~~~

## a hacker's guide to git

- http://wildlyinaccurate.com/a-hackers-guide-to-git

## rebase

`$ git rebase master foo`

With the format git rebase <base> <target>, the rebase command will take all of the commits from <target> and play them on top of <base> one by one

## 修改 commit message

- http://stackoverflow.com/questions/179123/edit-an-incorrect-commit-message-in-git

### 实际例子

~~~bash
git commit --amend 或者 git commit --amend -m "New commit message"

Changing the message of a commit that you've already pushed to your remote branch

git push <remote> <branch> --force

# Or

git push <remote> <branch> -f

~~~

## 配置

- `git config -l` 列出当前 config
- /etc/gitconfig 文件：系统中对所有用户都普遍适用的配置。若使用 git config 时用 --system 选项，读写的就是这个文件。
- ~/.gitconfig 文件：用户目录下的配置文件只适用于该用户。若使用 git config 时用 --global 选项，读写的就是这个文件。
- 当前项目的 Git 目录中的配置文件（也就是工作目录中的 .git/config 文件）：这里的配置仅仅针对当前项目有效。每一个级别的配置都会覆盖上层的相同配置，所以 .git/config 里的配置会覆盖 /etc/gitconfig 中的同名变量。

### 实际例子

#### ~/.gitconfig

~~~bash
[user]
  name = baya
  email = kayak.jiang@gmail.com
[alias]
  st = status
  ci = commit
  br = branch
  co = checkout
  df = diff
  dc = diff --cached
  lg = log -p
  who = shortlog -s --

[core]
  quotepath = off
[i18n]
  logoutputencoding = utf8
  commitencoding = utf8
[push]
   default = simple
~~~

#### ogproj/.git/config

~~~bash
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:MySite/ogproj.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
[branch "v5"]
	remote = origin
	merge = refs/heads/v5
[user]
        name = Jim
        email = tank@mysite.com
		
~~~

## 基础命令

### 创建分支

`$ git checkout -b 'new_branch'`

## stash

- `git stash list`

- `git stash drop stash@{0}`

- `git stash pop`

## 删除远程分支

`$ git push origin :newfeature`

## 删除本地分支

`$ git branch -d newfeature`

## 撤销 merge

`$ git merge --abort`

## fetch remote branch

- http://stackoverflow.com/questions/9537392/git-fetch-remote-branch

`$ git checkout --track origin/daves_branch`

## get remote branch

~~~bash
$ git fetch
$ git checkout --track remote_branch
~~~

## 回滚到某个 commit

- http://stackoverflow.com/questions/4114095/revert-to-a-previous-git-commit

`$ git reset --hard 0d1d7fc32`

## 合并远程分支

`$ git fetch`

`$ git merge remote_br_xxxx`
