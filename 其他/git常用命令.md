
分支切换注意第一次切换使用 `git checkout -b [newBranch]` 这里只是在本地创建了一个分支并且切换到了这个分支上，注意这里切换的时候一定要先切换到 `master` 上去切换到 `master` 上去之后我们创建分支就是以 `master` 为基准了然后再将本地这个分支和远程的分支联系起来

`git checkout [分支]` 切换到一个本地不存在到分之需要加上【-b】参数，表示创建一个新分支 这里要注意：在`IDEA`中我们`checkout`一个新分支后使用`【CTRL+T】`更新到时候需要先对此分支进行跟踪，使用命令： `git branch --set-upstream-to=origin/release_20200225 release_20200225`，如果不行，那还需要执行 `git fetch`

删除本地分支 `git branch -d 【分支】`

删除远程分支 `git push origin --delete 【分支】`

删除追踪分支 通过指令`git branch --delete --remotes <remote>/<branch>`,可以删除追踪分支,该操作并没有真正删除远程分支, 而是删除的本地分支和远程分支的关联关系,即追踪分支。如`git branch --rd origin/X`

查看当前分支 `git branch`

回退

-   1、`commit`了，但是还未`push` `git reset --soft [上个版本的commit_id]` 这里`--soft`表示回退保留相关代码，如果是`--hard`则不保留。注意：是上个版本的提交`id`
    
-   2、`commit`了，同时已经`push`了 这个很危险
    
强行回退到某个 commit
```
git reset --hard xxx
```

分支合并 `https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E6%96%B0%E5%BB%BA%E4%B8%8E%E5%90%88%E5%B9%B6`

创建分支

```
git branch new-branch
git checkout new-branch
git fetch
git push origin new-branch
git branch --set-upstream-to=origin/new-branch new-branch
```

当修改还未提交remote，但是此时分支已经被锁了，需要提交到另外一个分支上 使用`git stash`暂存 然后切换到另外一个分支，再使用 `git stash pop`取出暂存，然后提交 参考：`https://www.cnblogs.com/tocy/p/git-stash-reference.html` 这里不要`commit` `git stash list`查看暂存列表 `git stash drop stash@{xx}`删除某个暂存

本地工程和远程仓库建立关联 参考： `https://www.jianshu.com/p/142d4142721e`

1、本地工程首先要初始化：`git init`

2、在 `github` 建立同名仓库, 当然本地还需要

```
git add .
git commit -m "init"
```

3、建立两者联系： `git remote add origin https://github.com/yjaal/spring-security.git`

4、拉取：`git pull origin master` 这里如果本地或者远程仓库有同名文件冲突，可以先删除其中一个。若本地提交过一个文件，或者远程仓库也提交过一个文件， `git` 认为这是两个分支，此时：`git pull gitee master --allow-unrelated-histories`， `--allow-unrelated-histories` 会允许关联两个分支的历史分支。
最新的需要使用命令



5、提交：`git push --set-upstream github master`

```
git push -u origin "main"
```

查看本地仓库当前关联的远程地址: `git remote`


