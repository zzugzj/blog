---
title: Git基础
auther: gzj
tags:
  - git
  - 学习笔记
categories: git
description: >-
  介绍几个最基本的，也是最常用的 Git
  命令，以后绝大多数时间里用到的也就是这几个命令。使用这些命令，你就能初始化一个新的代码仓库，做一些适当配置；开始或停止跟踪某些文件；暂存或提交某些更新。我们还会展示如何让
  Git
  忽略某些文件，或是名称符合特定模式的文件；如何既快且容易地撤消犯下的小错误；如何浏览项目的更新历史，查看某两次更新之间的差异；以及如何从远程仓库拉数据下来或者推数据上去。
abbrlink: 1a60696b
date: 2021-01-04 10:34:56
---

## 取得项目的Git仓库

有两种取得 Git 项目仓库的方法。第一种是在现存的目录下，通过导入所有文件来创建新的 Git 仓库。第二种是从已有的 Git 仓库克隆出一个新的镜像仓库来。

### 在工作目录中初始化新仓库

要对现有的某个项目开始用 Git 管理，只需到此项目所在的目录，执行：

```
$ git init
```

初始化后，在当前目录下会出现一个名为 .git 的目录，所有 Git 需要的数据和资源都存放在这个目录中。

如果当前目录下有几个文件想要纳入版本控制，需要先用 `git add` 命令告诉 Git 开始对这些文件进行跟踪，然后提交：

```
$ git add *.c
$ git add README
$ git commit -m 'initial project version'
```

### 克隆现有的仓库

如果想对某个开源项目出一份力，可以先把该项目的 Git 仓库复制一份出来，这就需要用到 `git clone` 命令。实际上，即便服务器的磁盘发生故障，用任何一个克隆出来的客户端都可以重建服务器上的仓库，回到当初克隆时的状态。

克隆仓库的命令格式为 `git clone [url]`。比如，要克隆 Ruby 语言的 Git 代码仓库 Grit，可以用下面的命令：

```
$ git clone git://github.com/schacon/grit.git
```

这会在当前目录下创建一个名为`grit`的目录，其中包含一个 `.git` 的目录，用于保存下载下来的所有版本记录，然后从中取出最新版本的文件拷贝。如果希望在克隆的时候，自己定义要新建的项目目录名称，可以在上面的命令末尾指定新的名字：

```
$ git clone git://github.com/schacon/grit.git mygrit
```

唯一的差别就是，现在新建的目录成了 `mygrit`，其他的都和上边的一样。

Git 支持许多数据传输协议。之前的例子使用的是 `git://` 协议，不过你也可以用 `http(s)://` 或者 `user@server:/path.git` 表示的 SSH 传输协议。

## 记录每次更新到仓库

工作目录下面的所有文件都不外乎这两种状态：已跟踪或未跟踪。已跟踪的文件是指本来就被纳入版本控制管理的文件，在上次快照中有它们的记录，工作一段时间后，它们的状态可能是未更新，已修改或者已放入暂存区。而所有其他文件都属于未跟踪文件。它们既没有上次更新时的快照，也不在当前的暂存区域。初次克隆某个仓库时，工作目录中的所有文件都属于已跟踪文件，且状态为未修改。

在编辑过某些文件之后，Git 将这些文件标为已修改。我们逐步把这些修改过的文件放到暂存区域，直到最后一次性提交所有这些暂存起来的文件，如此重复。

![Git 下文件生命周期图。](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/2015-10-12_561bcb568ace5.png)

### 检查当前文件状态

要确定哪些文件当前处于什么状态，可以用 `git status` 命令。如果在克隆仓库之后立即执行此命令，会看到类似这样的输出：

```shell
$ git status
On branch master
nothing to commit, working tree clean
```

### 跟踪新文件

使用命令 `git add` 开始跟踪一个新文件。

所以，要跟踪 README 文件，运行：

```
$ git add README
```

此时再运行 `git status` 命令，会看到 README 文件已被跟踪，并处于暂存状态：

```shell
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   README
```

只要在 “Changes to be committed” 这行下面的，就说明是已暂存状态。如果此时提交，那么该文件此时此刻的版本将被留存在历史记录中。

### 暂存已修改文件

现在我们修改下之前已跟踪过的文件 `README.md`，然后再次运行 `status` 命令，会看到这样的状态报告：

```shell
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```

文件 `README.md` 出现在 “Changes not staged for commit” 这行下面，说明已跟踪文件的内容发生了变化，但还没有放到暂存区。要暂存这次更新，需要运行 `git add` 命令（这是个多功能命令，根据目标文件的状态不同，此命令的效果也不同：可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态等）。现在让我们运行 `git add` 将`README.md` 放到暂存区，然后再看看 `git status` 的输出：

```
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   README.md
```

### 状态简览

`git status` 命令的输出十分详细，但其用语有些繁琐。如果你使用 `git status -s` 命令或 `git status --short` 命令，你将得到一种更为紧凑的格式输出。运行 `git status -s` ，状态报告输出如下：

```
$ git status -s
 M README
MM Rakefile
A  lib/git.rb
M  lib/simplegit.rb
?? LICENSE.txt
```

新添加的未跟踪文件前面有 `??` 标记。

新添加到暂存区中的文件前面有 `A` 标记。

修改过的文件前面有 `M` 标记。

 `M` 有两个可以出现的位置，出现在右边的 `M` 表示该文件被修改了但是还没放入暂存区，出现在靠左边的 `M` 表示该文件被修改了并放入了暂存区。

### 忽略文件

一般我们总会有些文件无需纳入 Git 的管理，也不希望它们总出现在未跟踪文件列表。通常都是些自动生成的文件，比如日志文件，或者编译过程中创建的临时文件等。我们可以创建一个名为 `.gitignore` 的文件，列出要忽略的文件模式。来看一个实际的例子：

```
$ cat .gitignore
# Compiled class file
*.class
```

第一行告诉 Git 忽略所有以 `.class` 结尾的文件。一般这类对象文件和存档文件都是编译过程中出现的，我们用不着跟踪它们的版本。要养成一开始就设置好 `.gitignore` 文件的习惯，以免将来误提交这类无用的文件。

文件 `.gitignore` 的格式规范如下：

- 所有空行或者以注释符号 `＃` 开头的行都会被 Git 忽略。
- 可以使用标准的 glob 模式匹配。
- 匹配模式最后跟反斜杠（`/`）说明要忽略的是目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（`!`）取反。

所谓的 glob 模式是指 shell 所使用的简化了的正则表达式。星号（`*`）匹配零个或多个任意字符；`[abc]` 匹配任何一个列在方括号中的字符（这个例子要么匹配一个 a，要么匹配一个 b，要么匹配一个 c）；问号（`?`）只匹配一个任意字符；如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配（比如 `[0-9]` 表示匹配所有 0 到 9 的数字）。使用两个星号（`*`) 表示匹配任意中间目录，比如`a/**/z` 可以匹配 `a/z`, `a/b/z` 或 `a/b/c/z`等。

我们再看一个 `.gitignore` 文件的例子：

```shell
# 此为注释 – 将被 Git 忽略
# 忽略所有 .a 结尾的文件
*.a
# 但 lib.a 除外
!lib.a
# 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
/TODO
# 忽略 build/ 目录下的所有文件
build/
# 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
doc/*.txt
```

### 查看已暂存和未暂存的更新

实际上 `git status` 的显示比较简单，仅仅是列出了修改过的文件，如果要查看具体修改了什么地方，可以用 `git diff` 命令。`git diff` 会使用文件补丁的格式显示具体添加和删除的行。

要查看尚未暂存的文件更新了哪些部分，不加参数直接输入 `git diff`：

```
$ git diff
diff --git a/README.md b/README.md
index e69de29..f2ba8f8 100644
--- a/README.md
+++ b/README.md
@@ -0,0 +1 @@
+abc
\ No newline at end of file
```

此命令比较的是工作目录中当前文件和暂存区域快照之间的差异，也就是修改之后还没有暂存起来的变化内容。

若要看已经暂存起来的文件和上次提交时的快照之间的差异，可以用 `git diff --cached` 命令。（Git 1.6.1 及更高版本还允许使用 `git diff --staged`，效果是相同的，但更好记些。）来看看实际的效果：

```
$ git diff --staged
diff --git a/README.md b/README.md
index e69de29..f2ba8f8 100644
--- a/README.md
+++ b/README.md
@@ -0,0 +1 @@
+abc
\ No newline at end of file
```

### 提交更新

每次准备提交前，先用 `git status` 看下，是不是都已暂存起来了，然后再运行提交命令 `git commit`。

这种方式会启动文本编辑器以便输入本次提交的说明。另外也可以用 -m 参数后跟提交说明的方式，在一行命令中提交更新。

```
$ git commit
[master ee80182] test
 1 file changed, 1 insertion(+)
```

可以看到，提交后它会告诉你，当前是在哪个分支（master）提交的，本次提交的完整 SHA-1 校验和是什么（ee80182），以及在本次提交中，有多少文件修订过，多少行添改和删改过。

提交时记录的是放在暂存区域的快照，任何还未暂存的仍然保持已修改状态，可以在下次提交时纳入版本管理。每一次运行提交操作，都是对你项目作一次快照，以后可以回到这个状态，或者进行比较。

### 跳过使用暂存区域

使用暂存区域的方式可以精心准备要提交的细节，但有时候这么做略显繁琐。Git 提供了一个跳过使用暂存区域的方式，只要在提交的时候，给 `git commit` 加上 `-a` 选项，Git 就会自动把所有已经跟踪过的文件暂存起来一并提交，从而跳过 `git add` 步骤：

```shell
$ git commit -a -m "test"
[master 70da55d] test
 1 file changed, 2 insertions(+), 1 deletion(-)
```

### 移除文件

要从已跟踪文件清单中移除（确切地说，是从暂存区域移除），然后提交。可以用 `git rm` 命令完成此项工作，并连带从工作目录中删除指定的文件，这样以后就不会出现在未跟踪文件清单中了。

如果删除之前已经放到暂存区域后又修改了的话，则必须要用强制删除选项 `-f`（译注：即 force 的首字母），以防误删除文件后丢失修改的内容。

```shell
$ git rm qwe.txt
error: the following file has staged content different from both the
file and the HEAD:
    qwe.txt
(use -f to force removal)

Mr.Gzj@DESKTOP-GNL5LT6 MINGW64 /d/gitProject/offer (master)
$ git rm -f qwe.txt
rm 'qwe.txt'
```

另外一种情况是，我们想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。换句话说，仅是从跟踪清单中删除。比如一些大型日志文件或者一堆 `.a` 编译文件，不小心纳入仓库后，要移除跟踪但不删除文件，以便稍后在 `.gitignore` 文件中补上，用 `--cached` 选项即可：

```
$ git rm --cached readme.txt
```

后面可以列出文件或者目录的名字，也可以使用 glob 模式。比方说：

```
$ git rm log/\*.log
```

注意到星号 `*` 之前的反斜杠 `\`，因为 Git 有它自己的文件模式扩展匹配方式，所以我们不用 shell 来帮忙展开（译注：实际上不加反斜杠也可以运行，只不过按照 shell 扩展的话，仅仅删除指定目录下的文件而不会递归匹配。上面的例子本来就指定了目录，所以效果等同，但下面的例子就会用递归方式匹配，所以必须加反斜杠。）。此命令删除所有 `log/` 目录下扩展名为 `.log` 的文件。类似的比如：

```
$ git rm \*~
```

会递归删除当前目录及其子目录中所有 `~` 结尾的文件。

### 移动文件

Git 并不跟踪文件移动操作。如果在 Git 中重命名了某个文件，仓库中存储的元数据并不会体现出这是一次改名操作。

要在 Git 中对文件改名，可以这么做：

```
$ git mv file_from file_to
```

## 查看提交历史

在提交了若干更新之后，又或者克隆了某个项目，想回顾下提交历史，可以使用 `git log` 命令查看。

```shell
$ git log
commit 015436195b0a7e55123792425fb8213249d0b07d (HEAD -> master)
Author: gzj1999 <1045643052@qq.com>
Date:   Sat Jul 18 11:10:21 2020 +0800

    修改删除订单

commit b578cd1a0a917e5b975917d3f95c5ce692f6538f
Merge: ab86b7a 4a5964f
Author: gzj1999 <1045643052@qq.com>
Date:   Fri Jul 17 22:04:48 2020 +0800
```

默认不用任何参数的话，`git log` 会按提交时间列出所有的更新，最近的更新排在最上面。每次更新都有一个 SHA-1 校验和、作者的名字和电子邮件地址、提交时间，最后缩进一个段落显示提交说明。

我们常用 `-p` 选项展开显示每次提交的内容差异，用 `-2` 则仅显示最近的两次更新：

```
$ git log -p -2
commit 015436195b0a7e55123792425fb8213249d0b07d (HEAD -> master)
Author: gzj1999 <1045643052@qq.com>
Date:   Sat Jul 18 11:10:21 2020 +0800

    修改删除订单

diff --git a/src/main/java/com/clothes/controller/OrderController.java b/src/main/java/com/clothes/controller/OrderController.java
index 87295b1..d0f4c77 100644
--- a/src/main/java/com/clothes/controller/OrderController.java
+++ b/src/main/java/com/clothes/controller/OrderController.java
@@ -92,10 +92,14 @@ public class OrderController {
             return new Response("400", "没有此订单！");
         }
```

在做代码审查，或者要快速浏览其他协作者提交的更新都作了哪些改动时，就可以用这个选项。此外，还有许多摘要选项可以用，比如 `--stat`，仅显示简要的增改行数统计：

```
$ git log --stat
commit 015436195b0a7e55123792425fb8213249d0b07d (HEAD -> master)
Author: gzj1999 <1045643052@qq.com>
Date:   Sat Jul 18 11:10:21 2020 +0800

    修改删除订单

 src/main/java/com/clothes/controller/OrderController.java   | 4 ++++
 src/main/java/com/clothes/service/UserService.java          | 7 +++++++
 src/main/java/com/clothes/service/impl/UserServiceImpl.java | 7 +++++++
 3 files changed, 18 insertions(+)
```

还有个常用的 `--pretty` 选项，可以指定使用完全不同于默认格式的方式展示提交历史。比如用 `oneline` 将每个提交放在一行显示，这在提交数很大时非常有用。

一些其他常用的选项及其释义:

```
选项 说明
    -p 按补丁格式显示每个更新之间的差异。
    --stat 显示每次更新的文件修改统计信息。
    --shortstat 只显示 --stat 中最后的行数修改添加移除统计。
    --name-only 仅在提交信息后显示已修改的文件清单。
    --name-status 显示新增、修改、删除的文件清单。
    --abbrev-commit 仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。
    --relative-date 使用较短的相对时间显示（比如，“2 weeks ago”）。
    --graph 显示 ASCII 图形表示的分支合并历史。
    --pretty 使用其他格式显示历史提交信息。可用的选项包括 oneline，short，full，fuller 和 format（后跟指定格式）。
```

当 oneline 或 format 与另一个 `log` 选项 `--graph` 结合使用时尤其有用。这个选项添加了一些ASCII字符串来形象地展示你的分支、合并历史：

```
$ git log --pretty=format:"%h %s" --graph
*   1ebd4d5 up
|\
| * d121dae qw
* | a87e37d qw
|/
*   6c3f1d7 Merge branch 'iss53'
|\
| * 2519576 l
* | c7badcf up
```

### 限制输出长度

 `-<n>` 选项，其中的 `n` 可以是任何自然数，表示仅显示最近的若干条提交。不过实践中我们是不太用这个选项的，Git 在输出所有提交时会自动调用分页程序（less），要看更早的更新只需翻到下页即可。

另外还有按照时间作限制的选项，比如 `--since` 和 `--until`。下面的命令列出所有最近两周内的提交：

```
$ git log --since=2.weeks
```

你可以给出各种时间格式，比如说具体的某一天（“2008-01-15”），或者是多久以前（“2 years 1 day 3 minutes ago”）。

另一个真正实用的`git log`选项是路径(path)，如果只关心某些文件或者目录的历史提交，可以在 `git log` 选项的最后指定它们的路径。因为是放在最后位置上的选项，所以用两个短划线（`--`）隔开之前的选项和后面限定的路径名。

其他常用的类似选项。

```
选项 说明
-(n) 仅显示最近的 n 条提交
--since, --after 仅显示指定时间之后的提交。
--until, --before 仅显示指定时间之前的提交。
--author 仅显示指定作者相关的提交。
--committer 仅显示指定提交者相关的提交。
```

## 撤销操作

### 修改最后一次提交

有时候我们提交完了才发现漏掉了几个文件没有加，或者提交信息写错了。想要撤消刚才的提交操作，可以使用 `--amend` 选项重新提交：

```shell
$ git commit --amend #也可以 git commit --amend -m ""
```

此命令将使用当前的暂存区域快照提交。如果刚才提交完没有作任何改动，直接运行此命令的话，相当于有机会重新编辑提交说明，但将要提交的文件快照和之前的一样。

### 取消已经暂存的文件

有两个修改过的文件，我们想要分开提交，但不小心用 `git add .` 全加到了暂存区域。可以使用`git reset HEAD <file>...`的方式取消暂存。

- `git reset -–hard`：**彻底回退到某个版本**，本地的源码也会变为上一个版本的内容，撤销的commit中所包含的更改被冲掉。

### 取消对文件的修改

`git checkout -- <file>...`命令可以使文件恢复到修改前的版本，这条命令有些危险，所有对文件的修改都没有了，因为我们刚刚把之前版本的文件复制过来重写了此文件。所以在用这条命令前，请务必确定真的不再需要保留刚才的修改。

## 远程仓库的使用

远程仓库是指托管在网络上的项目仓库，可能会有好多个，其中有些你只能读，另外有些可以写。同他人协作开发某个项目时，需要管理这些远程仓库，以便推送或拉取数据，分享各自的工作进展。管理远程仓库的工作，包括添加远程库，移除废弃的远程库，管理各式远程库分支，定义是否跟踪这些分支，等等。

### 查看当前的远程库

要查看当前配置有哪些远程仓库，可以用 `git remote` 命令，它会列出每个远程库的简短名字。在克隆完某个项目后，至少可以看到一个名为 origin 的远程库，Git 默认使用这个名字来标识你所克隆的原始仓库：

```
$ git remote
origin
```

也可以加上 `-v` 选项（译注：此为 `--verbose` 的简写，取首字母），显示对应的克隆地址：

```
$ git remote -v
origin  https://gitee.com/gzj1999/test.git (fetch)
origin  https://gitee.com/gzj1999/test.git (push)
```

### 添加远程仓库

有时候可能在GitHub建一个仓库，Gitee建一个仓库。

要添加一个新的远程仓库，可以指定一个简单的名字，以便将来引用，运行 `git remote add [shortname] [url]`：

```
$ git remote add pb git@gitee.com:gzj1999/testgzj.git
```

现在可以用字符串 `pb` 指代对应的仓库地址了。比如说，要抓取所有 Paul 有的，但本地仓库没有的信息，可以运行 `git fetch pb`：

```
$ git fetch pb
warning: no common commits
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 948 bytes | 4.00 KiB/s, done.
From gitee.com:gzj1999/testgzj
 * [new branch]      master     -> pb/master
```

现在，testgzj的主干分支（master）已经完全可以在本地访问了，对应的名字是 `pb/master`。

### 从远程仓库抓取数据

正如之前所看到的，可以用下面的命令从远程仓库抓取数据到本地：

```
$ git fetch [remote-name]
```

此命令会到远程仓库中拉取所有你本地仓库中还没有的数据。运行完成后，你就可以在本地访问该远程仓库中的所有分支，将其中某个分支合并到本地，或者只是取出某个分支，一探究竟。

如果是克隆了一个仓库，此命令会自动将远程仓库归于 origin 名下。所以，`git fetch origin` 会抓取从你上次克隆以来别人上传到此远程仓库中的所有更新（或是上次 fetch 以来别人提交的更新）。有一点很重要，需要记住，**fetch 命令只是将远端的数据拉到本地仓库，并不自动合并到当前工作分支，只有当你确实准备好了，才能手工合并。**

### 推送数据到远程仓库

将本地仓库中的数据推送到远程仓库。实现这个任务的命令很简单： `git push [remote-name] [branch-name]`。如果要把本地的 master 分支推送到 `origin` 服务器上（再次说明下，克隆操作会自动使用默认的 master 和 origin 名字），可以运行下面的命令：

```
$ git push origin master
```

只有在所克隆的服务器上有写权限，或者同一时刻没有其他人在推数据，这条命令才会如期完成任务。如果在你推数据前，已经有其他人推送了若干更新，那你的推送操作就会被驳回。你必须先把他们的更新抓取到本地，合并到自己的项目中，然后才可以再次推送。

### 使用git的squash减少commit记录

假如git log如下：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/e7cd7b899e510fb35476621c2881b291d0430c00.png)

现在，我们希望将最后三个commit压缩为一个，这样push的时候也不至于太多无用的commit，可以使用`git rebase -i HEAD~3`来修改，这时候我们会发现进入编辑界面，并且显示内容如下：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/908fa0ec08fa513d73252bd4cddf2fffb3fbd960.png)

这个界面是让我们告诉git该如何处理每个commit。这里我们想保留f392171这个commit，所以我们需要做的就是将以下两个commit合并到第一个上，我们将编辑界面的内容改成这样即可：

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/63d0f703918fa0ec7d9f01cbd62523ea3f6ddbd7.png)

ok了，接下来esc，:wq保存即可了。

注意：不要合并已经push的commit。

### 查看远程仓库信息

我们可以通过命令 `git remote show [remote-name]` 查看某个远程仓库的详细信息，比如要看所克隆的 `origin` 仓库，可以运行：

```
$ git remote show origin
* remote origin
  Fetch URL: https://gitee.com/gzj1999/test.git
  Push  URL: https://gitee.com/gzj1999/test.git
  HEAD branch: master
  Remote branches:
    feature/sss                      new (next fetch will store in remotes/origin)
    master                           tracked
    refs/remotes/origin/feature/test stale (use 'git remote prune' to remove)
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```

它显示了有哪些远端分支还没有同步到本地（译注：feature/sss 分支），哪些已同步到本地的远端分支在远端服务器上已被删除（译注：refs/remotes/origin/feature/test分支），Local branch configured for 'git pull'意思是git pull默认拉取remote master分支，Local ref configured for 'git push':意思是默认push到master。

### 远程仓库的删除和重命名

Git 中可以用 `git remote rename` 命令修改某个远程仓库在本地的简称，比如想把 `pb` 改成 `paul`，可以这么运行：

```
$ git remote rename pb phub
$ git remote
origin
phub
```

注意，对远程仓库的重命名，也会使对应的分支名称发生变化，原来的 `pb/master` 分支现在成了 `phub/master`。

碰到远端仓库服务器迁移，或者原来的克隆镜像不再使用，又或者某个参与者不再贡献代码，那么需要移除对应的远端仓库，可以运行 `git remote rm` 命令：

```
$ git remote rm phub
$ git remote
origin
```

## 打标签

Git 可以对某一时间点上的版本打上标签。人们在发布某个软件版本（比如 v1.0 等等）的时候，经常这么做。

### 列显已有的标签

（下面用的 Git 自身项目仓库）列出现有标签的命令非常简单，直接运行 `git tag` 即可：

```
$ git tag
gitgui-0.10.0
gitgui-0.10.1
```

显示的标签按字母顺序排列，所以标签的先后并不表示重要程度的轻重。

我们可以用特定的搜索模式列出符合条件的标签。如果你只对 1.4.2 系列的版本感兴趣，可以运行下面的命令：

```
$ git tag -l v2.9.*
v2.9.0
v2.9.0-rc0
v2.9.0-rc1
v2.9.0-rc2
v2.9.1
v2.9.2
v2.9.3
v2.9.4
v2.9.5
```

### 新建标签

Git 使用的标签有两种类型：轻量级的（lightweight）和含附注的（annotated）。轻量级标签就像是个不会变化的分支，实际上它就是个指向特定提交对象的引用。而含附注标签，实际上是存储在仓库中的一个独立对象，它有自身的校验和信息，包含着标签的名字，电子邮件地址和日期，以及标签说明，标签本身也允许使用 GNU Privacy Guard (GPG) 来签署或验证。一般我们都建议使用含附注型的标签，以便保留相关信息；当然，如果只是临时性加注标签，或者不需要旁注额外信息，用轻量级标签也没问题。

#### 含附注的标签

创建一个含附注类型的标签非常简单，用 `-a` （译注：取 `annotated` 的首字母）指定标签名字即可：

```
$ git tag -a v1.4 -m 'my version 1.4'
$ git tag
v0.1
v1.3
v1.4
```

而 `-m` 选项则指定了对应的标签说明，Git 会将此说明一同保存在标签对象中。如果没有给出该选项，Git 会启动文本编辑软件供你输入标签说明。

可以使用 `git show` 命令查看相应标签的版本信息，并连同显示打标签时的提交对象。

#### 签署标签

如果你有自己的私钥，还可以用 GPG 来签署标签，只需要把之前的 `-a` 改为 `-s` （译注： 取 `signed` 的首字母）即可：

```
$ git tag -s v1.5 -m 'my signed 1.5 tag'
You need a passphrase to unlock the secret key for
user: "Scott Chacon <schacon@gee-mail.com>"
1024-bit DSA key, ID F721C45A, created 2009-02-09
```

#### 轻量级标签

轻量级标签实际上就是一个保存着对应提交对象的校验和信息的文件。要创建这样的标签，一个 `-a`，`-s` 或 `-m` 选项都不用，直接给出标签名字即可：

```
$ git tag v1.4-lw
$ git tag
v0.1
v1.3
v1.4
v1.4-lw
```

#### 验证标签

可以使用 `git tag -v [tag-name]` （译注：取 `verify` 的首字母）的方式验证已经签署的标签。此命令会调用 GPG 来验证签名，所以你需要有签署者的公钥，存放在 keyring 中，才能验证：

```
$ git tag -v v1.4.2.1
```

### 后期加注标签

你甚至可以在后期对早先的某次提交加注标签。比如在下面展示的提交历史中：

```
$ git log --pretty=oneline
a11c575af93aacd1eb019fa00c46cfdd55415314 (origin/feature/test) update qwe.txt.
dd186697004a318be67fdafb6577dfda16a077e8 ceshis
72adbf074a6037d61e0c28cf6e4de542d1b7e7d1 qwe
cd2830cfbf553e77c1889ab3163e37ad31f10970 test
```

我们忘了在提交 “ceshis” 后为此项目打上版本号 v1.2，没关系，现在也能做。只要在打标签的时候跟上对应提交对象的校验和（或前几位字符）即可：

```
$ git tag -a v1.1 dd186697
$ git tag
v1.1
```

### 分享标签

默认情况下，`git push` 并不会把标签传送到远端服务器上，只有通过显式命令才能分享标签到远端仓库。其命令格式如同推送分支，运行 `git push origin [tagname]` 即可：

```
$ git push origin v1.1
```

如果要一次推送所有本地新增的标签上去，可以使用 `--tags` 选项：

```
$ git push origin --tags
```