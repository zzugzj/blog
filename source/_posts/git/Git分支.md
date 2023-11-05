---
title: Git分支
auther: gzj
tags:
  - git
  - 学习笔记
categories:
  - git
description: >-
  几乎每一种版本控制系统都以某种形式支持分支。使用分支意味着你可以从开发主线上分离开来，然后在不影响主线的同时继续工作。在很多版本控制系统中，这是个昂贵的过程，常常需要创建一个源代码目录的完整副本，对大型项目来说会花费很长时间。
abbrlink: 7a7ff038
date: 2021-01-05 09:14:09
---

## 分支简介

Git 保存的不是文件差异或者变化量，而只是一系列文件快照。

在 Git 中提交时，会保存一个提交（commit）对象，该对象包含一个指向暂存内容快照的指针，包含本次提交的作者等相关附属信息，包含零个或多个指向该提交对象的父对象指针：首次提交是没有直接祖先的，普通提交有一个祖先，由两个或多个分支合并产生的提交则有多个祖先。

当使用 `git commit` 新建一个提交对象前，Git 会先计算每一个子目录（本例中就是项目根目录）的校验和，然后在 Git 仓库中将这些目录保存为树（tree）对象。之后 Git 创建的提交对象，除了包含相关提交信息以外，还包含着指向这个树对象（项目根目录）的指针，如此它就可以在将来需要的时候，重现此次快照的内容了。

使用`git commit`创建一个提交对象后，仓库中各个对象保存的数据和相互关系如图所示：

![首次提交对象及其树结构.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5aaaabe.png)

作些修改后再次提交，那么这次的提交对象会包含一个指向上次提交对象的指针（译注：即下图中的 parent 对象）。两次提交后，仓库历史会变成 的样子：

![提交对象及其父对象.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5ac6fec.png)

Git 中的分支，其实本质上仅仅是个指向 commit 对象的可变指针。Git 会使用 master 作为分支的默认名字。在若干次提交后，你其实已经有了一个指向最后一次提交对象的 master 分支，它在每次提交的时候都会自动向前移动。

![分支及其提交历史.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5adcb57.png)

新建一个 testing 分支，可以使用 `git branch` 命令：

```
$ git branch testing
```

这会在当前 commit 对象上新建一个分支指针。

![两个指向相同提交历史的分支。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5aee442.png)

Git通过保存一个名为HEAD的特别指针来知道当前在哪个分支上工作，HEAD是一个指向你正在工作中的本地分支的指针。运行 `git branch` 命令，仅仅是建立了一个新的分支，但不会自动切换到这个分支中去，所以在这个例子中，我们依然还在 master 分支里工作。

![HEAD 指向当前所在的分支.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5b0bb09.png)

要切换到其他分支，可以执行 `git checkout` 命令。我们现在转换到新建的 testing 分支：

```
$ git checkout testing
```

这样 HEAD 就指向了 testing 分支。

![HEAD 指向当前所在的分支.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5b1b5c1.png)

如果现在再提交一次，就会变成如下图所示：

![HEAD 分支随着提交操作自动向前移动.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5b31055.png)

现在testing 分支向前移动了一格，而 master 分支仍然指向原先 `git checkout` 时所在的 commit 对象。现在我们回到 master 分支看看：

```
$ git checkout master
```

这条命令做了两件事。它把 HEAD 指针移回到 master 分支，并把工作目录中的文件换成了 master 分支所指向的快照内容。

我们作些修改后再次提交的话，项目的提交历史就会产生分叉。刚才我们创建了一个分支，转换到其中进行了一些工作，然后又回到原来的主分支进行了另外一些工作。这些改变分别孤立在不同的分支里：我们可以在不同分支里反复切换，并在时机成熟时把它们合并到一起。

![项目分叉历史.](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5b52163.png)

Git 中的分支实际上仅是一个包含所指对象校验和（40 个字符长度 SHA-1 字串）的文件，所以创建和销毁一个分支就变得非常廉价。

## 分支的新建与合并

### 分支的新建与切换

要新建并切换到该分支，运行 `git checkout` 并加上 `-b` 参数：

```
$ git checkout -b iss53
Switched to a new branch "iss53"
```

这相当于执行下面这两条命令：

```
$ git branch iss53
$ git checkout iss53
```

不过在此之前，暂存区或者工作目录里那些还没有提交的修改会和即将检出的分支产生冲突从而阻止 Git 切换分支。切换分支的时候最好保持一个清洁的工作区域。

分支如果要进行合并可以使用`git merge` 命令来进行合并：

![基于  分支的紧急问题分支（hotfix branch）。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5ccb01a.png)

```
$ git checkout master
$ git merge hotfix
Updating 5c2482d..15dcd1c
Fast-forward
 qwe.txt | 1 +
 1 file changed, 1 insertion(+)
```

合并时出现了“Fast forward”的提示。由于当前 `master` 分支所在的提交对象是要并入的 `hotfix` 分支的直接上游，Git 只需把 `master` 分支指针直接右移。换句话说，如果顺着一个分支走下去可以到达另一个分支的话，那么 Git 在合并两者时，只会简单地把指针右移，因为这种单线的历史分支不存在任何需要解决的分歧，所以这种合并过程可以称为快进（Fast forward）。

由于当前 `hotfix` 分支和 `master` 都指向相同的提交对象，所以 `hotfix` 已经完成了历史使命，可以删掉了。使用 `git branch` 的 `-d` 选项执行删除操作：

```
$ git branch -d hotfix
Deleted branch hotfix (was 15dcd1c).
```

### 分支的合并

在问题 #53 相关的工作完成之后，可以合并回 `master` 分支。实际操作同前面合并 `hotfix` 分支差不多，只需回到 `master` 分支，运行 `git merge` 命令指定要合并进来的分支：

```
$ git checkout master
$ git merge iss53
Merge made by the 'recursive' strategy.
 pom.xml | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

这次合并操作的底层实现，并不同于之前 `hotfix` 的并入方式。因为这次你的开发历史是从更早的地方开始分叉的。由于当前 `master` 分支所指向的提交对象（C4）并不是 `iss53` 分支的直接祖先，Git 不得不进行一些额外处理。就此例而言，Git 会用两个分支的末端（C4 和 C5）以及它们的共同祖先（C2）进行一次简单的三方合并计算。图 3-16 用红框标出了 Git 用于合并的三个提交对象：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/18333fig0316-tn.png)

这次，Git 没有简单地把分支指针右移，而是对三方合并后的结果重新做一个新的快照，并自动创建一个指向它的提交对象（C6）（见图 3-17）。这个提交对象比较特殊，它有两个祖先（C4 和 C5）。

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/18333fig0317-tn.png)

### 遇到冲突时的分支合并

有时候合并操作并不会如此顺利。如果在不同的分支中都修改了同一个文件的同一部分，Git 就无法干净地把两者合到一起（译注：逻辑上说，这种问题只能由人来裁决。）。如果你在解决问题 #53 的过程中修改了 `hotfix` 中修改的部分，将得到类似下面的结果：

```
$ git merge iss54
Auto-merging qwe.txt
CONFLICT (content): Merge conflict in qwe.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Git 作了合并，但没有提交，它会停下来等你解决冲突。要看看哪些文件在合并时发生冲突，可以用 `git status` 查阅:

```
$ git status
On branch master
Your branch is ahead of 'origin/master' by 5 commits.
  (use "git push" to publish your local commits)

You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   qwe.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

在Unmerged paths里列出所有冲突文件。

打开qwe.txt可以看到：

```
啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊
<<<<<<< HEAD
亲委
=======
士大夫撒旦
>>>>>>> iss54
```

可以看到 `=======` 隔开的上半部分，是 `HEAD`（即 `master` 分支，在运行 `merge` 命令时所切换到的分支）中的内容，下半部分是在 `iss54` 分支中的内容。解决冲突的办法无非是二者选其一或者由你亲自整合到一起。比如你可以通过把这段内容替换为下面这样来解决：

```
啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊
亲委
士大夫撒旦
```

然后执行git add和git commit完成合并提交。

## 分支管理

`git branch` 命令不仅仅能创建和删除分支，如果不加任何参数，它会给出当前所有分支的清单：

```
$ git branch
  iss53
  iss54
* master
```

注意看 `master` 分支前的 `*` 字符：它表示当前所在的分支。也就是说，如果现在提交更新，`master` 分支将随着开发进度前移。若要查看各个分支最后一个提交对象的信息，运行 `git branch -v`：

```
$ git branch -v
  iss53  a87e37d qw
  iss54  d121dae qw
* master 1ebd4d5 [ahead 7] up
```

要从该清单中筛选出你已经（或尚未）与当前分支合并的分支，可以用 `--merge` 和 `--no-merged` 选项。比如用 `git branch --merge` 查看哪些分支已被并入当前分支：

```
$ git branch --merged
  iss53
  iss54
* master
```

一般来说，列表中没有 `*` 的分支通常都可以用 `git branch -d` 来删掉。原因很简单，既然已经把它们所包含的工作整合到了其他分支，删掉也不会损失什么。

另外可以用 `git branch --no-merged` 查看尚未合并的工作：

```
$ git branch --no-merged
    testing
```

它会显示还未合并进来的分支。由于这些分支中还包含着尚未合并进来的工作成果，所以简单地用 `git branch -d` 删除该分支会提示错误，因为那样做会丢失数据：

```
$ git branch -d testing
error: The branch 'testing' is not fully merged.
If you are sure you want to delete it, run 'git branch -D testing'.
```

不过，如果你确实想要删除该分支上的改动，可以用大写的删除选项 `-D` 强制执行，就像上面提示信息中给出的那样。

## 利用分支进行开发的工作流程

### 长期分支

许多使用 Git 的开发者都喜欢用这种方式来开展工作，比如仅在 `master` 分支中保留完全稳定的代码，即已经发布或即将发布的代码。与此同时，他们还有一个名为 `develop` 或 `next` 的平行分支，专门用于后续的开发，或仅用于稳定性测试 — 当然并不是说一定要绝对稳定，不过一旦进入某种稳定状态，便可以把它合并到 `master` 里。

![渐进稳定分支的工作流（“silo”）视图。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5de1972.png)

### 特性分支

在任何规模的项目中都可以使用特性（Topic）分支。一个特性分支是指一个短期的，用来实现单一特性或与其相关工作的分支。

现在我们来看一个实际的例子。请看图 ，由下往上，起先我们在 `master` 工作到 C1，然后开始一个新分支 `iss91` 尝试修复 91 号缺陷，提交到 C6 的时候，又冒出一个解决该问题的新办法，于是从之前 C4 的地方又分出一个分支 `iss91v2`，干到 C8 的时候，又回到主干 `master` 中提交了 C9 和 C10，再回到 `iss91v2` 继续工作，提交 C11，接着，又冒出个不太确定的想法，从 `master` 的最新提交 C10 处开了个新的分支 `dumbidea` 做些试验。

![拥有多个特性分支的提交历史。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5df1ada.png)

现在，假定两件事情：我们最终决定使用第二个解决方案，即 `iss91v2` 中的办法；另外，我们把 `dumbidea` 分支拿给同事们看了以后，发现它竟然是个天才之作。所以接下来，我们准备抛弃原来的 `iss91` 分支（实际上会丢弃 C5 和 C6），直接在主干中并入另外两个分支。最终的提交历史将变成下图这样：

![合并了  和  分支之后的提交历史。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5e18de5.png)

请务必牢记这些分支全部都是本地分支，这一点很重要。当你在使用分支及合并的时候，一切都是在你自己的 Git 仓库中进行的 — 完全不涉及与服务器的交互。

## 远程分支

远程分支（remote branch）是对远程仓库中的分支的索引。它们是一些无法移动的本地分支；只有在 Git 进行网络交互时才会更新。

远程引用是对远程仓库的引用（指针），包括分支、标签等等。你可以通过 `git ls-remote (remote)` 来显式地获得远程引用的完整列表，或者通过 `git remote show (remote)` 获得远程分支的更多信息。然而，一个更常见的做法是利用远程跟踪分支。

我们用 `(远程仓库名)/(分支名)` 这样的形式表示远程分支。比如我们想看看上次同 `origin` 仓库通讯时 `master` 分支的样子，就应该查看 `origin/master` 分支。如果你和同伴一起修复某个问题，但他们先推送了一个 `iss53` 分支到远程仓库，虽然你可能也有一个本地的 `iss53` 分支，但指向服务器上最新更新的却应该是 `origin/iss53` 分支。

> ### “origin” 并无特殊含义

> 远程仓库名字 “origin” 与分支名字 “master” 一样，在 Git 中并没有任何特别的含义一样。同时 “master” 是当你运行 `git init` 时默认的起始分支名字，原因仅仅是它的广泛使用，“origin” 是当你运行 `git clone` 时默认的远程仓库名字。如果你运行 `git clone -o booyah`，那么你默认的远程分支名字将会是 `booyah/master`。

如果你在本地 `master` 分支做了些改动，与此同时，其他人向 `git.ourcompany.com` 推送了他们的更新，那么服务器上的 `master` 分支就会向前推进，而于此同时，你在本地的提交历史正朝向不同方向发展。不过只要你不和服务器通讯，你的 `origin/master` 指针仍然保持原位不会移动。

![本地与远程的工作可以分叉。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5f813c8.png)

可以运行 `git fetch origin` 来同步远程服务器上的数据到本地。该命令首先找到 `origin` 是哪个服务器（本例为 `git.ourcompany.com`），从上面获取你尚未拥有的数据，更新你本地的数据库，然后把 `origin/master` 的指针移到它最新的位置上。

![ 更新你的远程仓库引用。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb5f98f62.png)

### 推送本地分支

把本地分支推送到远程仓库`git push (远程仓库名) (分支名)`。

运行 `git push origin serverfix:serverfix` ，它的意思是“上传我本地的 serverfix 分支到远程仓库中去，仍旧称它为 serverfix 分支”。通过此语法，你可以把本地分支推送到某个命名不同的远程分支：若想把远程分支叫作 `awesomebranch`，可以用 `git push origin serverfix:awesomebranch` 来推送数据。

`git checkout -b serverfix origin/serverfix`是会切换到新建的 `serverfix` 本地分支，其内容同远程分支 `origin/serverfix` 一致。

### 跟踪远程分支

在克隆仓库时，Git 通常会自动创建一个名为 `master` 的分支来跟踪 `origin/master`。这正是 `git push` 和 `git pull` 一开始就能正常工作的原因。当然，你可以随心所欲地设定为其它跟踪分支，比如 `origin` 上除了 `master` 之外的其它分支。刚才我们已经看到了这样的一个例子：`git checkout -b [分支名] [远程名]/[分支名]`。如果你有 1.6.2 以上版本的 Git，还可以用 `--track` 选项简化：

```
$ git checkout --track origin/serverfix
```

要为本地分支设定不同于远程分支的名字，只需在第一个版本的命令里换个名字：

```
$ git checkout -b sf origin/serverfix
```

现在你的本地分支 `sf` 会自动将推送和抓取数据的位置定位到 `origin/serverfix` 了。

如果想要查看设置的所有跟踪分支，可以使用 `git branch` 的 `-vv` 选项。这会将所有的本地分支列出来并且包含更多的信息，如每一个分支正在跟踪哪个远程分支与本地分支是否是领先、落后或是都有。

```
$ git branch -vv
* master  df81349 [origin/master: ahead 8] as
  testing 22add77 qwe
```

这里可以看到` master`分支正在跟踪 `origin/master`并且 “ahead” 是 8，意味着本地有两个提交还没有推送到服务器上。

如果想要统计最新的领先与落后数字，需要在运行此命令前抓取所有的远程仓库。可以像这样做：`$ git fetch --all; git branch -vv`

### 删除远程分支

`git push origin --delete 分支名`

在删除远程分支时，同名的本地分支并不会被删除，所以还需要单独删除本地同名分支。

## 变基

在 Git 中整合来自不同分支的修改主要有两种方法：`merge` 以及 `rebase`。

### 变基的基本操作

假设分支提交历史如下：

![分叉的提交历史](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb6162ea5.png)

如果使用merge命令，它会把两个分支的最新快照C3和C4以及二者共同祖先C2合并，生成一个新的快照并提交。

其实，还有一种方法：你可以提取在 `C4` 中引入的补丁和修改，然后在 `C3` 的基础上再应用一次。在 Git 中，这种操作就叫做 *变基*。你可以使用 `rebase` 命令将提交到某一分支上的所有修改都移至另一分支上，就好像“重新播放”一样。

在上面这个例子中，运行：

```
$ git checkout experiment
$ git rebase master
Successfully rebased and updated refs/heads/experiment.
```

它的原理是首先找到这两个分支（即当前分支 `experiment`、变基操作的目标基底分支 `master`）的最近共同祖先 `C2`，然后对比当前分支相对于该祖先的历次提交，提取相应的修改并存为临时文件，然后将当前分支指向目标基底 `C3`, 最后以此将之前另存为临时文件的修改依序应用。

![将  中的修改变基到  上](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb6189bbc.png)

现在回到 `master` 分支，进行一次快进合并。

```
$ git checkout master
$ git merge experiment
```

![master 分支的快进合并](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb619d7bf.png)

### 变基的风险

**不要对在你的仓库外有副本的分支执行变基。**

变基操作的实质是丢弃一些现有的提交，然后相应地新建一些内容一样但实际上不同的提交。如果你已经将提交推送至某个仓库，而其他人也已经从该仓库拉取提交并进行了后续工作，此时，如果你用 `git rebase` 命令重新整理了提交并再次推送，你的同伴因此将不得不再次将他们手头的工作与你的提交进行整合，如果接下来你还要拉取并整合他们修改过的提交，事情就会变得一团糟。