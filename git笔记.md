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

> 注意这个`--global`是设置全局git的信息。
>
> 如果**只**修改当前这个项目的用户名信息「比如其他的项目是公司的项目，这一个项目是开源项目。使用了不同的用户名和密码」，不加`--global`即可。

### 撤销init

```bash
# 撤销git init
rm -rf .git
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
git reset HEAD -- 文件名
```

```bash
# 从工作区检测撤回文件修改
# 这个--很重要。如果不加--，就变成了「切换到另一分支」命令
git checkout -- 文件名
```

> `git checkout -- 文件名`是为了撤销修改，目标文件是会回滚到原状态的。
>
> 而`git rm --cached 文件名`只是不提交到暂存区。
>
> ![image-20210924144110451](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210924144110451.png)
>
> ```bash
> # 从暂存区撤回「但本地文件系统保留对文件修改」
> git rm --cached <文件>…
> ```
>
> <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210924144223461.png" alt="image-20210924144223461" style="zoom:50%;" />



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
git checkout -- 文件名
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

> 退出`git log`和`git reflog`，按`q`即可。

### 版本描述修改

> 也就是`git commit -m '错误的信息描述'`，想要修改这个「错误的信息描述」。

有两种方式

- 一种方式是从「版本库」回退到「暂存区」，然后重新提交

  ```bash
  git reset --soft HEAD^
  git commit -m '正确的信息描述'
  ```

  > `HEAD^`标识上一个版本
  >
  > `HEAD^^`表示上两个版本「上上个版本」
  >
  > 也可以使用`HEAD~1`、`HEAD~2`的方式进行上面的操作。

- 另一种方式是「直接修改」

  ```bash
  git commit --amend
  ```

  进入当前版本描述，修改保存

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210924155350672.png" alt="image-20210924155350672" style="zoom:50%;" />

  保存退出即可。



## 分支管理

### 创建、查看、切换、删除

```bash
# 查看分支
git branch
```

```bash
# 创建分支
# 从哪个分支创建的分支，就会原样拷贝原分支
git branch 分支名
```

```bash
# 切换分支
git checkout 分支名
# 或者
git switch 分支名
```

```bash
# 创建并切换至分支
git branch -b 分支名
# 等价于
git branch 分支名
git checkout 分支名
```

```bash
# 删除分支
git branch -d 分支名
```

***

git有几种图示分支的方式

- `git log --graph --all`

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925214959557.png" alt="image-20210925214959557" style="zoom:50%;" />

  如果想要查看更加简介的版本可以增加`--oneline`参数

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925215145244.png" alt="image-20210925215145244" style="zoom:50%;" />

- 使用git gui

  ```bash
  gitk --all
  ```

  ![image-20210925215412806](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925215412806.png)

### 合并

分支合并分为三种方式merge、rebase、cherry-pick。下面将一一介绍。

#### merge

merge也有三种方式fast-forward、no-ff、squash。

目的是统一的：「比如A分支要去合并B分支的代码，会在A分支上产生**一个**新版本」。

但是三个却有不同，下面一一来看。

- **fast-forward**

  如果B分支「被合并分支」在在当前分支的下游「没有分叉」，默认使用快速合并。

  举例：在dev分支新提交文件dev.txt。

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925221856232.png" alt="image-20210925221856232"  />

  在master进行合并。

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925222328409.png" alt="image-20210925222328409"  />

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925222648536.png" alt="image-20210925222648536"  />

  这两个分支的id是一样的。在删除了dev分支之后，不会留下dev的记录「他就跟在master上提交了一个新版本一模一样。」

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925222918156.png" alt="image-20210925222918156"  />

  画张图标识就是这样。

  ![image-20210925223624257](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925223624257.png)

- **no-ff**

  ```bash
  git merge --no-ff branch1
  ```

  ![image-20210925224903267](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925224903267.png)

  下面这张图展示了合并过程。

  ![image-20210925224539575](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925224539575.png)

  它会在合并分支「master」上创建一个新合并版本「Merge：m3」。

  master的log是这样的：

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925224713304.png" alt="image-20210925224713304"  />

  <img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210925224809895.png" alt="image-20210925224809895"  />

  **branch1分支删除之后，master的记录依然存在。**

- **squash**

  这个选项就是把之前的分支合并操作分为两步：①先**把待分支代码拉取到本地**。②在合并分支自行提交。

  ![image-20210926092141782](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926092141782.png)

  提交完成之后，只有master的一条记录「branch1合并记录不保留」。

  ![image-20210926092350309](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926092350309.png)

  graph图中能看到这个合并操作是断开的「分支`4c1b25c`和master `a8a3742`没有连接」。

  ![image-20210926092602588](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926092602588.png)

  squash的流程是下面这样：

  ![image-20210926093112397](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926093112397.png)

  可以理解为master拉取了`d1`分支的提交代码，以master的身份进行了提交「当然这里如果相对记录进行继续操作，当然可以」。

#### rebase

rebase的合并方式稍有不同，不止可以合并分支代码，它还包括了本地代码的版本合并「**同分支多版本合并**」。

##### 同分支多版本合并

假设有一个大功能，每天都会提交一点点代码。但是想在代码提交时只显示一个融合版本。

![image-20210926094910426](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926094910426.png)

> 目前在分支上就有一个功能的多次提交记录。

现在对这三个记录进行合并。

```bash
# 对当前分支的最近三个版本进行合并
git rebase -i HEAD~3
```

- 第一步就是三个版本的代码融合

  ![image-20210926095544404](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926095544404.png)

  以第一个版本作为提交版本，其余「pick改为s的版本」追加到之前的版本。

- 然后对三个版本设置一个统一的提交版本

  ![image-20210926095924618](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926095924618.png)

  保存就提交。

![image-20210926100030242](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926100030242.png)

可以在log中看到之前的三条提交版本合并为一个版本。

![image-20210926100215341](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926100215341.png)

##### 不同分支合并

<img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926112527070.png" alt="image-20210926112527070" style="zoom:50%;" />

最初，从`m2`版本开始，出现了不同的两个分支`m3「master」`和`d1「dev」`。下面进行合并。

- 先在待合并分支「dev」进行**变基**操作。

  ```bash
  # 切换到待合并分支
  git checkout dev
  # 变基「也就是把dev的分叉节点改到master的m3」
  git rebase master
  ```

- 第二步就是**冲突解决**。

  > 我这里模拟了一个冲突。
  >
  > dev分支「branch1」上之前的d1版本是「**9b105b** function01完整代码」
  >
  > 又在master上提交了新的修改版本「**80837f5** master_function_01」

  ![image-20210926105733871](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926105733871.png)

  在rebase的时候因为`branch1`文件产生了冲突。按照提示执行即可。

  - 手动解决冲突。
  - `git add/rm <confilicted_files>`
  - `git rebase --continue`

  如果要放弃这次变基操作，可以执行`git rebase --abort`

  经过这一步之后branch1就完成了rebase「基节点改到了`m3「80837f5 master_function_01」`」。

  而之前的d1版本「**9b105b** function01完整代码」也发生了改变。

  ![image-20210926112112898](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926112112898.png)

  「**9b105b** function01完整代码」变成了「**f298175** function01完整代码」

  ![image-20210926112603093](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926112603093.png)

- 最后一步是**合并**。

  ```bash
  # 切换回合并分支
  git checkout master
  # 进行合并「fast-forward」
  git merge branch1
  ```

  ![image-20210926113101579](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926113101579.png)

#### cherry-pick

它的作用就是讲指定的提交哈希应用于当前分支。

```bash
git cherry-pick <commitHash>
```

比如，一个人同时开发不同功能模块，创建了两个分支。但是在一些基础模块可以复用。那么可以使用cherry-pick把基础模块提交的commitHash合并到本地分支。

![image-20210926134027607](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926134027607.png)

`d1`需要优先于`d2`提交。

也可以一次pick多个提交

```bash
# HashA 和 HashB 一起pick
git cherry-pick <HashA> <HashB>
# 提交HashA到HashB之间的所有 (HashA,HashB]
git cherry-pick HashA..HashB
# 如果要pick[HashA,HashB]闭区间
git cherry-pick HashA^..HashB
```

cherry-pick有一些常用配置项。

- `-e`/`--edit`

  打开外部编辑器，编辑提交信息

- `-n`/`--no-commit`

  只更新本地文件和暂存区，不产生新提交。

- `-x`

  在提交信息的末尾追加一行「cherry picked from commit ...」，方便后续追踪。

- `-s`/`--signoff`

  在提交信息的末尾追加一行操作者的签名。

- `-m`/`--mainline`

  如果commit是一个合并节点「来自于两个分支的合并」，cherry-pick默认会失败，因为不知道应该采用哪个分支的代码改动。

  ```bash
  # cherry-pick采用commitHash中编号1的分支改动
  git cherry-pick -m 1 <commitHash>
  ```

  一般来说1号标识接收改动的分支，2号分支是被合并分支「改动来源」。

当发生冲突后，cherry-pick跟rebase的操作方式是一样的。

- cherry-pick会先停下来。

- 解决冲突之后，手动将文件加入暂存区`git add <conflictFile>`。

  然后`git cherry-pick --continue`让它继续。

- 或者采用`--abort`放弃合并，文件还原。

- 或者`--quit`，退出cherry-pick。但不还原文件。

### 工作区暂存

> 工作区暂存和暂存区不一样的。

工作区暂存的目的是：目前手头上的工作尚未完成，需要决绝其他问题「比如线上bug」。

先看一下命令列表

|                命令                 | 说明                                                         |
| :---------------------------------: | :----------------------------------------------------------- |
| **git stash** <save "save message"> | 执行存储时，添加备注，方便查找，只有git stash 也要可以的，但查找时不方便识别。 |
|         **git stash list**          | 查看stash了哪些存储 按「q」退出                              |
|         **git stash show**          | 显示做了哪些改动，默认show第一个存储,如果要显示其他存贮，后面加stash@{$num}，比如第二个 git stash show stash@{1} |
|        **git stash show -p**        | 显示第一个存储的改动，如果想显示其他存存储，命令：git stash show stash@{$num} -p ，比如第二个：git stash show stash@{1} -p |
|         **git stash apply**         | 应用某个存储,但不会把存储从存储列表中删除，默认使用第一个存储,即stash@{0}，如果要使用其他个，git stash apply stash@{$num} ， 比如第二个：git stash apply stash@{1} |
|          **git stash pop**          | 命令恢复之前缓存的工作目录，将缓存堆栈中的对应stash删除，并将对应修改应用到当前的工作目录下,默认为第一个stash,即stash@{0}，如果要应用并删除其他stash，命令：git stash pop stash@{$num} ，比如应用并删除第二个：git stash pop stash@{1} |
|  **git stash drop **stash@{\$num}   | 丢弃stash@{\$num}存储，从列表中删除这个存储                  |
|         **git stash clear**         | 删除所有缓存的stash                                          |

### 远程协同

> 前面的版本库是本地的版本库，它只存在于我的电脑里。
>
> 当团队中的不同成员进行协同开发时，就会进行远程系统。
>
> 他需要一个远程的代码仓库『而不是只在我的电脑上』，比如github、gitee或者公司/团队自建的gitlib……

比如我就在gitee上创建了远程仓库「用于git操作实战」

![image-20210926141944483](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926141944483.png)

第一步就是**把本地的代码库和远程代码库进行关联**。

```bash
# origin可以自己命名，可以理解为远程库的代称
git remote add origin https://gitee.com/wangigor/git_test.git
```

关联之后可以使用

```bash
# 查看远程库的信息
git remote
# 查看远程库的详细信息
git remote -v
```

![image-20210926142800044](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926142800044.png)

可以在本地分支上手动指定远程分支名称

```bash
git branch --set-upstream-tp=origin/<远程分支名> 本地分支名
```

默认两个分支名是一样的。

***

然后就是**把本地版本库推送到远程**。

> 比如要把master分支推送到远程master。

```bash
# 切换到master分支
git checkout master
# 本地版本库推送到远程
git push origin master
```

如果，其他人有提交过这个分支的代码，就会出现冲突。

先把远程库代码拉倒本地解决冲突然后提交。

***

从远程库同步到本地库。

```bash
# 把远程master分支拉倒当前分支的版本库
git pull origin master
# 或者 把远程版本库的全部分支更新都拉回本地
git pull origin
# 完整版
git pull <远程库名称> <远程分支名>:<本地分支名>
```

`git pull origin master`就等价于

```bash
git fetch origin master
git merge FETCH_HEAD
```

这个`FETCH_HEAD`就是远程HEAD，可以通过`git log -p FETCH_HEAD`查看。

![image-20210926144730280](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926144730280.png)



## 标签

> 标签呢。可以理解为`commitHash`的传播版本「昨天晚上上线的commitHash是「f0d6c5fb0a177f6d765ceca812aeaa033cbf547d」，肯定不好理解与传播。」，可以把当前一个版本标记为人类能够看懂或者方便记录的版本号。

![image-20210926150127585](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926150127585.png)

比如刚才的`f0d6c5fb0a177f6d765ceca812aeaa033cbf547d`就可以标记为`v1.0.0`版。

```bash
git tag -a v1.0.0
```

需要填写这个标签的描述信息。

![image-20210926150411193](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926150411193.png)

保存之后，就有了版本号。

![image-20210926150526306](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926150526306.png)

也可以通过`git tag`查看所有标签。

不过，到目前远程版本库还不知道有这一个tag信息的「`v1.0.0`这个tag只存在于本地」，需要手动推送。

```bash 
git push origin --tags
```

![image-20210926151017553](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926151017553.png)

![image-20210926151046348](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926151046348.png)

## 工作流 WorkFlow

![image-20210926161616455](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210926161616455.jpeg)



