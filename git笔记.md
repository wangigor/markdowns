# git

## 初始化

### 本地初始化

```bash
# 设置操作人
git config --global user.email "you@example.com"
git config --global user.name "your name"
# 创建工作空间
mkdir module_name
# 交由git管理
git init
```

## 工作区-暂存-版本库

> ==todo==示例截图

```bash
# 工作区到工作检测区
git status
# 版本库提交版本号查询
git log
# 版本库历史版本查询「能查到reset之后的记录」
git reflog
```

### 工作区-暂存

```bash
# 单个文件提交到暂存
git add 文件名
# 整个目录提交到暂存
git add .
# 匹配的文件提交到暂存
git add 表达式
```

```bash
# 暂存回到工作区检测
git reset HEAD --文件名
```

```bash
# 从工作区检测撤回文件修改
git checkout 文件名
```

### 暂存-版本库

```bash
# 暂存提交到版本库
git commit -m '版本描述'
```

```bash
# 版本库回滚到暂存
git reset --soft 版本号
```

### 工作区-暂存区-版本库

```bash
# 版本库回滚到工作区检测
git reset --mix 版本号
# 等价于
git reset --soft 版本号
git reset HEAD
```

```bash
# 版本库回滚到工作区
git reset --hard 版本号
# 等价于
git reset --mix 版本号
git checkout 文件名
```

举例：假设commit了两个版本到版本库`v1`和`v2`

- 想要从`v2`回退到`v1`

  ```bash
  # 查询v1版本号
  git log
  # 直接回滚到工作区
  git reset --hard 版本号
  ```

- 想要再从`v1`撤销回退到`v2`

  ```bash
  # 查询v2版本号
  git reflog
  # 按照版本号回滚到工作区
  git reset --hard 版本号
  ```



> git remote add 标识名(origin) 远程地址
>
> git clone 远程地址
>
> Git push origin master
>
> Git pull origin master
>
> Git branch
>
> git checkout 分支名
>
> git log 「--oneline」--graph
>
> git merge 分支 合并分支
>
> git merge 分支
>
> 



（1）**git stash** save "save message" : 执行存储时，添加备注，方便查找，只有git stash 也要可以的，但查找时不方便识别。

（2）**git stash list** ：查看stash了哪些存储 按「q」退出

（3）**git stash show** ：显示做了哪些改动，默认show第一个存储,如果要显示其他存贮，后面加stash@{$num}，比如第二个 git stash show stash@{1}

（4）**git stash show -p** : 显示第一个存储的改动，如果想显示其他存存储，命令：git stash show stash@{$num} -p ，比如第二个：git stash show stash@{1} -p

（5）**git stash apply** :应用某个存储,但不会把存储从存储列表中删除，默认使用第一个存储,即stash@{0}，如果要使用其他个，git stash apply stash@{$num} ， 比如第二个：git stash apply stash@{1} 

（6）**git stash pop** ：命令恢复之前缓存的工作目录，将缓存堆栈中的对应stash删除，并将对应修改应用到当前的工作目录下,默认为第一个stash,即stash@{0}，如果要应用并删除其他stash，命令：git stash pop stash@{$num} ，比如应用并删除第二个：git stash pop stash@{1}

（7）**git stash drop** stash@{\$num} ：丢弃stash@{\$num}存储，从列表中删除这个存储

（8）`**git stash clear** ：`删除所有缓存的stash



## 撤销commit

> 本地修改了文件，执行了
>
> ```bash
> git add file
> git commit -m "修改原因"
> ```
>
> 但是没有push，如何撤销commit

```bash
git reset --soft HEAD^
```

可以重置上一次commit操作，但是没有撤销add「没有删除工作空间的修改文件」，如果需要撤销add，可以使用--hard。

「HEAD^」表示上一个版本。也可以写成HEAD\~1。如果想要撤销两次commit，使用HEAD\~2。

```bash
git commit --amend
```

也可以使用这个命令进入vim直接修改。