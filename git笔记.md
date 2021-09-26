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
> ![image-20210924144110451](https://gitee.com/wangigor/typora-images/raw/master/image-20210924144110451.png)
>
> ```bash
> # 从暂存区撤回「但本地文件系统保留对文件修改」
> git rm --cached <文件>…
> ```
>
> <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210924144223461.png" alt="image-20210924144223461" style="zoom:50%;" />



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

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210924155350672.png" alt="image-20210924155350672" style="zoom:50%;" />

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

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925214959557.png" alt="image-20210925214959557" style="zoom:50%;" />

  如果想要查看更加简介的版本可以增加`--oneline`参数

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925215145244.png" alt="image-20210925215145244" style="zoom:50%;" />

- 使用git gui

  ```bash
  gitk --all
  ```

  ![image-20210925215412806](https://gitee.com/wangigor/typora-images/raw/master/image-20210925215412806.png)

### 合并

分支合并分为三种方式merge、rebase、cherry-pick。下面将一一介绍。

#### merge

merge也有三种方式fast-forward、no-ff、squash。

目的是统一的：「比如A分支要去合并B分支的代码，会在A分支上产生**一个**新版本」。

但是三个却有不同，下面一一来看。

- **fast-forward**

  如果B分支「被合并分支」在在当前分支的下游「没有分叉」，默认使用快速合并。

  举例：在dev分支新提交文件dev.txt。

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925221856232.png" alt="image-20210925221856232"  />

  在master进行合并。

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925222328409.png" alt="image-20210925222328409"  />

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925222648536.png" alt="image-20210925222648536"  />

  这两个分支的id是一样的。在删除了dev分支之后，不会留下dev的记录「他就跟在master上提交了一个新版本一模一样。」

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925222918156.png" alt="image-20210925222918156"  />

  画张图标识就是这样。

  ![image-20210925223624257](https://gitee.com/wangigor/typora-images/raw/master/image-20210925223624257.png)

- **no-ff**

  ```bash
  git merge --no-ff branch1
  ```

  ![image-20210925224903267](https://gitee.com/wangigor/typora-images/raw/master/image-20210925224903267.png)

  下面这张图展示了合并过程。

  ![image-20210925224539575](https://gitee.com/wangigor/typora-images/raw/master/image-20210925224539575.png)

  它会在合并分支「master」上创建一个新合并版本「Merge：m3」。

  master的log是这样的：

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925224713304.png" alt="image-20210925224713304"  />

  <img src="https://gitee.com/wangigor/typora-images/raw/master/image-20210925224809895.png" alt="image-20210925224809895"  />

  **branch1分支删除之后，master的记录依然存在。**

- **squash**

  这个选项就是把之前的分支合并操作分为两步：①先**把待分支代码拉取到本地**。②在合并分支自行提交。

  ![image-20210926092141782](https://gitee.com/wangigor/typora-images/raw/master/image-20210926092141782.png)

  提交完成之后，只有master的一条记录「branch1合并记录不保留」。

  ![image-20210926092350309](https://gitee.com/wangigor/typora-images/raw/master/image-20210926092350309.png)

  graph图中能看到这个合并操作是断开的「分支`4c1b25c`和master `a8a3742`没有连接」。

  ![image-20210926092602588](https://gitee.com/wangigor/typora-images/raw/master/image-20210926092602588.png)

  squash的流程是下面这样：

  ![image-20210926093112397](https://gitee.com/wangigor/typora-images/raw/master/image-20210926093112397.png)

  可以理解为master拉取了`d1`分支的提交代码，以master的身份进行了提交「当然这里如果相对记录进行继续操作，当然可以」。

#### rebase

rebase的合并方式稍有不同，不止可以合并分支代码，它还包括了本地代码的版本合并「**同分支多版本合并**」。

##### 同分支多版本合并

假设有一个大功能，每天都会提交一点点代码。但是想在代码提交时只显示一个融合版本。

![image-20210926094910426](https://gitee.com/wangigor/typora-images/raw/master/image-20210926094910426.png)

> 目前在分支上就有一个功能的多次提交记录。

现在对这三个记录进行合并。

```bash
# 对当前分支的最近三个版本进行合并
git rebase -i HEAD~3
```

- 第一步就是三个版本的代码融合

  ![image-20210926095544404](https://gitee.com/wangigor/typora-images/raw/master/image-20210926095544404.png)

  以第一个版本作为提交版本，其余「pick改为s的版本」追加到之前的版本。

- 然后对三个版本设置一个统一的提交版本

  ![image-20210926095924618](https://gitee.com/wangigor/typora-images/raw/master/image-20210926095924618.png)

  保存就提交。

![image-20210926100030242](https://gitee.com/wangigor/typora-images/raw/master/image-20210926100030242.png)

可以在log中看到之前的三条提交版本合并为一个版本。

![image-20210926100215341](https://gitee.com/wangigor/typora-images/raw/master/image-20210926100215341.png)



#### cherry-pick















（1）**git stash** save "save message" : 执行存储时，添加备注，方便查找，只有git stash 也要可以的，但查找时不方便识别。

（2）**git stash list** ：查看stash了哪些存储 按「q」退出

（3）**git stash show** ：显示做了哪些改动，默认show第一个存储,如果要显示其他存贮，后面加stash@{$num}，比如第二个 git stash show stash@{1}

（4）**git stash show -p** : 显示第一个存储的改动，如果想显示其他存存储，命令：git stash show stash@{$num} -p ，比如第二个：git stash show stash@{1} -p

（5）**git stash apply** :应用某个存储,但不会把存储从存储列表中删除，默认使用第一个存储,即stash@{0}，如果要使用其他个，git stash apply stash@{$num} ， 比如第二个：git stash apply stash@{1} 

（6）**git stash pop** ：命令恢复之前缓存的工作目录，将缓存堆栈中的对应stash删除，并将对应修改应用到当前的工作目录下,默认为第一个stash,即stash@{0}，如果要应用并删除其他stash，命令：git stash pop stash@{$num} ，比如应用并删除第二个：git stash pop stash@{1}

（7）**git stash drop** stash@{\$num} ：丢弃stash@{\$num}存储，从列表中删除这个存储

（8）**git stash clear** ：删除所有缓存的stash
