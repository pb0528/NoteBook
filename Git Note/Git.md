# Git 



## Git 基础 - 获取 Git仓库

### 获取Git 仓库

有两种取得Git 项目仓库的方法。第一种是在现有项目或目录下导入所有导入所有文件到Git中；第二种是从一个服务器克隆一个现有的Git仓库

### 在现有目录中初始化仓库

如果你打算使用Git 来对现有的项目进行管理，你只需要进入改项目目录并输入：

> $ git init

如果你是在一个已经存在文件的文件夹（而不是空文件夹）中初始化Git仓库来进行版本控制的话，你应该开始跟踪这些文件并提交。你可通过`git add` 命令来实现对指定文件的跟踪，然后执行`git commit`提交：

> 1.  $ git add *.c
> 2.  $ git add LICENSE
> 3.  $ git commit -m 'initial project version'

### 克隆现有仓库

如果你想获得一份已经存在了的Git仓库的拷贝，这时就要用到`git clone` 命令。

> 格式 git clone [url]
>
> git clone https://git 

## 记录每次更新到仓库

![](C:\Users\PB\Desktop\git_test\PicResource\resource_1.png)

Figure 8. 文件的状态变化周期

### 检查当前文件状态

要查看哪些文件处于什么状态，可以用`git status`命令。如果在克隆仓库后立即使用此命令，会看到类似这样的输出：

```
1.$ git status
2.On branch master
3.nothing to commit, working directory clean
```

这说明你现在的工作目录相当干净。换句话说，所有的以跟踪文件在上次提交后都未被更改过。

```
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        hello.txt

nothing added to commit but untracked files present (use "git add" to track)

```

在状态报告可以看到新建的README 文件出现在`Untracked files`下面。

### 跟踪新文件

使用命令`git add`开始跟踪一个文件。所以，要跟踪README文件，运行：

> 1. $ git add README

此时再运行`git status`命令，会看到README文件已被跟踪，并处于暂存状态：

```
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   GitFileStatusCheck.vsdx
        new file:   PicResource/resource_1.png
        new file:   hello.txt
        new file:   ~$$GitFileStatusCheck.~vsdx
```

只要在`Changes to be committed`这行下面的，就说明是已暂存状态。如果此时提交，那么该文件此时此刻的版本将会被留存在历史文件中。你可能会想起之前我们使用`git  init`后就运行了`git add (file)`命令，开始跟踪当前目录下的文件。`git add`命令使用文件或目录的路径作为参数；如果参数是目录的路径，该命令将递归地跟踪改目录下的所有文件。

### 暂存已修改的文件

现在我们来修改一个已被跟踪地文件。如果你修改了一个文件，其已被跟踪然后运行`git status`命令，会看到如下内容：

```
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   Git.md
        new file:   GitFileStatusCheck.vsdx
        new file:   PicResource/resource_1.png
        new file:   hello.txt
        new file:   ~$$GitFileStatusCheck.~vsdx

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   Git.md
```

文件`Git.md`出现在`changes to be committed`这行下面，这说明已跟踪文件的内容发生变化，但是还没有放到暂存区下面，这说明已跟踪文件的内容发生了变化，但是还没有放到暂存区下面。要暂存这次更新，需要用到`git add`命令。这是个多功能的命令：可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把冲突文件标记为已解决状态等。将这个命令理解为”添加内容到下一次提交中“而不是”将一个文件添加的项目中“要更加合适。现在用`git add`将”Git.md“放到暂存区，然后再看看`git status`的输出：

```
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   Git.md
        new file:   GitFileStatusCheck.vsdx
        new file:   PicResource/resource_1.png
        new file:   hello.txt
        new file:   ~$$GitFileStatusCheck.~vsdx
```

如若任然继续要修改暂存区文件，仍需用git add 添加到暂存区中。

### 状态简览

`git status`命令输出十分详细，但其用于有些繁琐。如果你用 `git status -s`命令或`git status -short`命令，你将得到一种更为紧凑的格式输出。运行`git status -s`，状态报告输出如下：

```
$ git status -s
AM Git.md
A  GitFileStatusCheck.vsdx
A  PicResource/resource_1.png
A  hello.txt
A  ~$$GitFileStatusCheck.~vsdx
?? README.md
```

新加的为跟踪文件前面有`??`标记，新添加到暂存区的文件前面有`A`标记，修改的文件前面有`M`标记。

### 忽略文件

一般我们总会有些文件无须纳入Git的管理，也不希望它们总出现在为跟踪文件列表。通常都是些自动生成的文件，比如日志文件，或者编译过程中创建的临时文件等。在这种情况下，我们可以创建一个名为`.gitignore`的文件，列出要忽略的文件模式。来看一个实际例子：

```
$ cat .gitignore
*.[oa]
*~
```

第一行告诉Git忽略所有以`.o`或`.a`结尾的文件。一般这类对象文件和存档文件都是编译过程中出现的。第二行告诉Git忽略所有以波浪(~)结尾的文件，许多文件编辑软件都用这样的文件保存副本。

文件`.gitignore`格式规范如下：

* 所有空行或者#开头的行都会被Git忽略
* 可以使用标准的glob模式匹配
* 匹配模式可以以（/）开头防止递归
* 匹配模式可以以（/）结尾指定目录
* 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（!）取反。

GitHub 有一个十分详细的针对数十种项目及语言的`.gitignore`文件列表，可以在https://github.com/github/gitignore找到它。

### 查看已暂存和未暂存的修改

如果`git status`命令的输出对于你来说过于模式可以用`git diff`查看文件的差异

### 提交更新

现在暂存区域已经准备妥当可以准备提交了。在此之前，请一定要确认还有什么修改过的或者新建的文件还没有`git add`过，否则提交的时候不会记录这些还没暂存起来的变化。这些修改过的文件只保留在本地磁盘中。所以每次准备提交前，先用`git status`看下，是不是都以暂存 了，然后再运行提交命令`git commit`：

> $ git commit

也可以使用`git commit -m`添加message提交

### 跳过使用暂存区域

`git commit `加上`-a`选项，Git就会自动把所有已经跟踪过的文件暂存起来一并提交，从而跳过`git add`步骤

### 移除文件

要从Git中移除某个文件，就不需要从以跟踪文件清单中移除（确切地说，是从暂存区中移除），然后提交。可以用`git rm`命令完成此工作。

如果忘记添加`.gitignore`文件，不小心把一个很大的日志文件或者一堆`.a`这样地文件添加暂存区，这一做法尤其有用。

> $ git rm --cached README

`git rm `命令后面可以列出文件或者目录名字，也可以使用`glob`模式。

> $ git rm log/\*.log

### 移动文件

运行`git mv`命令先当于运行一下三条命令

```
$ mv README.md README
$ git rm README.md
$ git add README
```



