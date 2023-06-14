
```toc

```


## 分支

### 创建分支

```bash
git branch [branch_name]
```
创建分支一般需要切换到 master/main 分支上面。否则在哪个分支创建，就是以哪个分支为基础创建。

### 切换分支

```bash
git checkout [branch_name]
```

这里要注意：在 `IDEA` 中我们 `checkout` 一个新分支后使用 `【CTRL+T】` 更新到时候需要先对此分支进行跟踪，使用命令： `git branch --set-upstream-to=origin/release_20200225 release_20200225`，如果不行，那还需要执行 `git fetch`

### 创建并切换分支

```bash
git checkout -b [branch_name]
```

### 删除分支

```bash
# 删除本地分支
git branch -d [branch_name]

# 删除远程分支
git push origin --delete [branch_name]
```

### 查看分支

```bash
git branch
```

### 删除追踪分支

通过指令

```bash
git branch --delete --remotes <remote>/<branch>
```

可以删除追踪分支, 该操作并没有真正删除远程分支, 而是删除的本地分支和远程分支的关联关系, 即追踪分支。如 `git branch --rd origin/X`

### 回退

- 1、`commit` 了，但是还未 `push` `git reset --soft [上个版本的commit_id]` 这里 `--soft` 表示回退保留相关代码，如果是 `--hard` 则不保留。注意：是上个版本的提交 `id`
    
- 2、`commit` 了，同时已经 `push` 了这个很危险
    
强行回退到某个 commit

```bash
git reset --hard xxx
```

### 分支合并

`https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E6%96%B0%E5%BB%BA%E4%B8%8E%E5%90%88%E5%B9%B6`

```bash
# 一般先切换到main分支然后再进行合并
git merge [branch_name]
```

### 分支暂存

当修改还未提交 remote，但是此时分支已经被锁了，需要提交到另外一个分支上使用 `git stash` 暂存然后切换到另外一个分支，再使用 `git stash pop` 取出暂存，然后提交。

参考：`https://www.cnblogs.com/tocy/p/git-stash-reference.html` 这里不要 `commit` 

```bash
# 查看暂存列表
git stash list
# 删除某个缓存
git stash drop stash@{xx}
```

## 本地工程和远程仓库建立关联

参考： `https://www.jianshu.com/p/142d4142721e`

1、本地工程首先要初始化：`git init`

2、在 `github` 建立同名仓库, 当然本地还需要

```
git add .
git commit -m "init"
```

3、建立两者联系

```bash
git remote add origin https://github.com/yjaal/spring-security.git
```

这里的 origin 表示一个远程地址标签，没有其他意义，可以查看 config 文件。

4、拉取
```bash
git pull --rebase origin main
```

这里如果本地或者远程仓库有同名文件冲突，可以先删除其中一个。若本地提交过一个文件，或者远程仓库也提交过一个文件， `git` 认为这是两个分支，此时：`git pull gitee master --allow-unrelated-histories`， `--allow-unrelated-histories` 会允许关联两个分支的历史分支。

5、提交

```
git push -u origin "main"
```

查看本地仓库当前关联的远程地址: `git remote`

如果无法 push，可以在这里 `https://www.ipaddress.com/site/github.com` 查询下最新的 github 的 ip 地址，替换掉/etc/hosts 中的映射。

## Tag

### 列出标签

```bash
git tag
```

### 创建标签

Git 支持两种标签：轻量标签（lightweight）与附注标签（annotated）。

轻量标签很像一个不会改变的分支——它只是某个特定提交的引用。

而附注标签是存储在 Git 数据库中的一个完整对象，它们是可以被校验的，其中包含打标签者的名字、电子邮件地址、日期时间，此外还有一个标签信息，并且可以使用 GNU Privacy Guard （GPG）签名并验证。通常会建议创建附注标签，这样你可以拥有以上所有信息。但是如果你只是想用一个临时的标签，或者因为某些原因不想要保存这些信息，那么也可以用轻量标签。

#### 附注标签

在 Git 中创建附注标签十分简单。最简单的方式是当你在运行 `tag` 命令时指定 `-a` 选项：

```console
$ git tag -a v1.4 -m "my version 1.4"
$ git tag
v0.1
v1.3
v1.4
```

`-m` 选项指定了一条将会存储在标签中的信息。如果没有为附注标签指定一条信息，Git 会启动编辑器要求你输入信息。

通过使用 `git show` 命令可以看到标签信息和与之对应的提交信息：

```console
$ git show v1.4
tag v1.4
Tagger: Ben Straub <ben@straub.cc>
Date:   Sat May 3 20:19:12 2014 -0700

my version 1.4

commit ca82a6dff817ec66f44342007202690a93763949
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Mar 17 21:52:11 2008 -0700

    changed the version number
```

#### 轻量标签

另一种给提交打标签的方式是使用轻量标签。 轻量标签本质上是将提交校验和存储到一个文件中——没有保存任何其他信息。 创建轻量标签，不需要使用 `-a`、`-s` 或 `-m` 选项，只需要提供标签名字：

```console
$ git tag v1.4-lw
$ git tag
v0.1
v1.3
v1.4
v1.4-lw
v1.5
```

这时，如果在标签上运行 `git show`，你不会看到额外的标签信息。 命令只会显示出提交信息：

```console
$ git show v1.4-lw
commit ca82a6dff817ec66f44342007202690a93763949
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Mar 17 21:52:11 2008 -0700

    changed the version number
```

#### 后期打标签

你也可以对过去的提交打标签。 假设提交历史是这样的：

```console
$ git log --pretty=oneline
15027957951b64cf874c3557a0f3547bd83b3ff6 Merge branch 'experiment'
a6b4c97498bd301d84096da251c98a07c7723e65 beginning write support
0d52aaab4479697da7686c15f77a3d64d9165190 one more thing
6d52a271eda8725415634dd79daabbc4d9b6008e Merge branch 'experiment'
0b7434d86859cc7b8c3d5e1dddfed66ff742fcbc added a commit function
4682c3261057305bdd616e23b64b0857d832627b added a todo file
166ae0c4d3f420721acbb115cc33848dfcc2121a started write support
9fceb02d0ae598e95dc970b74767f19372d61af8 updated rakefile
964f16d36dfccde844893cac5b347e7b3d44abbc commit the todo
8a5cbc430f1a9c3d00faaeffd07798508422908a updated readme
```

现在，假设在 v1.2 时你忘记给项目打标签，也就是在 “updated rakefile” 提交。 你可以在之后补上标签。 要在那个提交上打标签，你需要在命令的末尾指定提交的校验和（或部分校验和）：

```console
$ git tag -a v1.2 9fceb02
```

可以看到你已经在那次提交上打上标签了：

```console
$ git tag
v0.1
v1.2
v1.3
v1.4
v1.4-lw
v1.5

$ git show v1.2
tag v1.2
Tagger: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Feb 9 15:32:16 2009 -0800

version 1.2
commit 9fceb02d0ae598e95dc970b74767f19372d61af8
Author: Magnus Chacon <mchacon@gee-mail.com>
Date:   Sun Apr 27 20:43:35 2008 -0700

    updated rakefile
...
```

#### push 标签

默认情况下，`git push` 命令并不会传送标签到远程仓库服务器上。 在创建完标签后你必须显式地推送标签到共享服务器上。 这个过程就像共享远程分支一样——你可以运行 `git push origin <tagname>`。

```console
$ git push origin v1.5
Counting objects: 14, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (12/12), done.
Writing objects: 100% (14/14), 2.05 KiB | 0 bytes/s, done.
Total 14 (delta 3), reused 0 (delta 0)
To git@github.com:schacon/simplegit.git
 * [new tag]         v1.5 -> v1.5
```

如果想要一次性推送很多标签，也可以使用带有 `--tags` 选项的 `git push` 命令。 这将会把所有不在远程仓库服务器上的标签全部传送到那里。

```console
$ git push origin --tags
Counting objects: 1, done.
Writing objects: 100% (1/1), 160 bytes | 0 bytes/s, done.
Total 1 (delta 0), reused 0 (delta 0)
To git@github.com:schacon/simplegit.git
 * [new tag]         v1.4 -> v1.4
 * [new tag]         v1.4-lw -> v1.4-lw
```

现在，当其他人从仓库中克隆或拉取，他们也能得到你的那些标签。
