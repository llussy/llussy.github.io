---
layout: post
title: "git使用"
date: 2019-12-30 16:40:30 +0800
catalog: ture
multilingual: false
tags:
    - linux
---

[TOC]

### git clone

git clone是把整个git项目拷贝下来，包括里面的日志信息，git项目里的分支，你也可以直接切换、使用里面的分支等等。

下载完成后可以进入目录，使用git branch查看有多少分支，使用git tag查看有多少tags。

**使用私钥克隆**
```bash
ssh -i ~/.ssh/id_rsa_example" git clone example
ssh-agent sh -c "ssh-add /Users/lisai/anfeng/id_rsa;git clone git@git.test.com:lisai/test-back.git"
```

**gitlab token clone**

```bash
git clone https://gitlab+deploy-token-7:Vop81UtiaZJ2xFooRGEf@git.test.com/llussy/Prometheus-WeChat-Alerts.git
```



### git fetch

**git fetch pb  从远程仓库获取数据，拉取仓库中有的但你没有的信息，它并不会自动合并或修改你当前的工作。**



### git pull

**git pull相当于git fetch和git merge。其意思是先从远程下载git项目里的文件，然后将文件与本地的分支进行merge。**

```bash
$ git pull <远程主机名> <远程分支名>:<本地分支名>
比如，取回origin主机的next分支，与本地的master分支合并，需要写成下面这样。

$ git pull origin next:master
如果远程分支是与当前分支合并，则冒号后面的部分可以省略。

$ git pull origin next
$ git pull origin master   (省略了本地分支)
git pull origin layer7 (拉取远程指定分支)
```

### git rebase
[git rebase介绍](https://zhuanlan.zhihu.com/p/75499871)
```bash
git remote add upstream https://github.com/xxx/xxx
git fetch upstream
git checkout master
git rebase upstream/master
git push -f origin master
```



### git push 

```bash
git push <远程主机名> <本地分支名>:<远程分支名>
```

注意，分支推送顺序的写法是<来源地>:<目的地>，所以git pull是<远程分支>:<本地分支>，而git push是<本地分支>:<远程分支>。

如果省略远程分支名，则表示将本地分支推送与之存在”追踪关系”的远程分支(通常两者同名)，如果该远程分支不存在，则会被新建。

```
$ git push origin master
```

**上面命令表示，将本地的master分支推送到origin主机的master分支。如果后者不存在，则会被新建。**

如果省略本地分支名，则表示删除指定的远程分支，因为这等同于推送一个空的本地分支到远程分支。

```bash
$ git push origin :master
# 等同于
$ git push origin --delete master
```

上面命令表示删除origin主机的master分支。

如果当前分支与远程分支之间存在追踪关系，则本地分支和远程分支都可以省略。

```bash
$ git push origin
```

上面命令表示，将当前分支推送到origin主机的对应分支。

如果当前分支只有一个追踪分支，那么主机名都可以省略。

```bash
$ git push
```

如果当前分支与多个主机存在追踪关系，则可以使用-u选项指定一个默认主机，这样后面就可以不加任何参数使用git push。



```bash
$ git push --all origin
```

上面命令表示，将所有本地分支都推送到origin主机。



如果远程主机的版本比本地版本更新，推送时Git会报错，要求先在本地做git pull合并差异，然后再推送到远程主机。这时，如果你一定要推送，可以使用–force选项。

```bash
$ git push --force origin
```

上面命令使用–force选项，结果导致在远程主机产生一个”非直进式”的合并(non-fast-forward merge)。除非你很确定要这样做，否则应该尽量避免使用–force选项。



最后，git push不会推送标签(tag)，除非使用–tags选项。

```bash
$ git push origin --tags
```

有时候当远程xxx分支被删掉了后，用git branch -a 你还可以看到本地还有remote/origin/xxx这个分支，那么你可以使用git fetch -p 这个命令可以帮你同步最新的远程分支，并删掉本地被删了的远程分支。

### git remote

git remote 列出已经存在的远程分支  -v 详细信息

 添加远程仓库 git remote add <shortname> <url>

git remote add pb <https://github.com/paulboone/ticgit>

### git checkout

**恢复到修改前的状态**

```bash
# 恢复暂存区的指定文件到工作区  修改文件后，没有add 执行git check [file]恢复到修改前状态
$ git checkout [file]

# 恢复暂存区的所有文件到工作区
$ git checkout .
```

**创建分支并切换**

```bash
$ git checkout -b test master 
取master分支并创建test 分支（master和test一致）
相当于git branch test && git checkout test
```

**使用某次commit创建分支**

```bash
git checkout -b lisai eaeb8f771be34793c9ade030629167bc537d12af
```

### git diff

git diff 修改之后还没有暂存起来的变化内容。

git diff --cached 或 git diff --staged 若要查看已暂存的将要添加到下次提交里的内容



### git log

**git log 查看提交历史**

**参数：**

-p 选项展开显示每次提交的内容差异

-2 只显示最近两次更新

--stat 仅显示简要的增改行数统计



### git status

**git status -s**

```bash
$ git status -s
 M README        ###修改过没有放入暂存区的文件
MM Rakefile
A lib/git.rb      ###新加到暂存区的文件
M lib/simplegit.rb  ###修改过并放入暂存区的文件
?? LICENSE.txt     ###未跟踪文件
```

新添加的未跟踪文件前面有 ?? 标记

新添加到暂存区中的文件前面有 A 标记

修改过的文件前面有 M 标记。你可能注意到了 M有两个可以出现的位置，出现在右边的 M 表示该文件被修改了但是还没放入暂存区，出现在靠左边的 M表示该文件被修改了并放入了暂存区。 



### git revert

revert 之后你的本地代码会回滚到指定的历史版本,这时你再 git push 就可以把线上的代码更新.

revert 使用,需要先找到你想回滚版本唯一的commit标识代码,可以用 git log 或者在adgit搭建的web环境历史提交记录里查看.

```
git revert c011eb3c20ba6fb38cc94fe5a8dda366a3990c61
```

通常,前几位即可

```
git revert c011eb3
```

git revert是用一次新的commit来回滚之前的commit，**git reset是直接删除指定的commit**

看似达到的效果是一样的,其实完全不同.



### git reset 

```bash
git reset --hard HEAD^    # 回退到上一版本 --hard 本地修改也会被清除，彻底还原
git reset --soft HEAD^    # 仅仅重置HEAD到制定的版本，不会修改index和working tree
git reset --hard commit_id
```

Git必须知道当前版本是哪个版本，在Git中，`用HEAD表示当前版本`，也就是最新的提交，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100。

**git reset HEAD file可以把暂存区的修改撤销掉（unstage），重新放回工作区**

### .gitlab-ci.yml

**golang**

```yml
stages:
  - build

build:
  stage: build
  image: docker.test.com/base/golang:1.12.6
  script:
    - export GO111MODULE=on
    - export GOPROXY=https://goproxy.io
    - cd cmd/configine && go build -v -mod=readonly

```



```yaml
image: golang:1.12

cache:
  paths:
    - /apt-cache
    - /go/pkg/mod/cache/download

stages:
  - test
  - build

before_script:
  - export GO111MODULE=on
  - export GOPROXY=https://goproxy.io
  - go get  github.com/swaggo/swag/cmd/swag@v1.5.1

unit_tests:
  stage: test
  script:
    - make test

build:
  stage: build
  script:
    - make

```


