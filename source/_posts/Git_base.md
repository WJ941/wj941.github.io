---
title: Git基础
date: 2018-08-18 08:49:30
tags:
---

### Git工作区和文件的标识
#### Git区域
Git有2个区域：**working directory**, **staging area**。

**working directory** : 本地编辑区域

**staging area** : 通过`git add`命令把本地编辑过或者新的文件的存放到**staging area**, 准备后续commit
#### Git里文件的标识

**working directory** 里文件状态标识是对应staging area，2种状态： **untracked**, **modified**：

**untracked** : 在 **working directory**里新建的文件，还没有通过`git add`添加到 **staging area**，在命令行运行`git status`后如下图所示，
{% asset_img untracked_file.png %}

**modified** : 文件之前已经通过`git add`添加到 **staging area**，本地修改后还未添加到**staging area**标记为*modified*，在命令行运行`git status`后如下图所示，
{% asset_img modified_file.png %}

**staging area** 里文件状态标识是对应最近的commit，有3种状态： *new file*， *modified*, *deleted*

### Git文件生命周期
{% asset_img lifecycle.png %}
工作区的文件分为2个状态：**tracked**和 **untracked**

**tracked** 指在staging area或者commit snapshot中的文件，总之是经过Git处理过的文件,其中又分为 **unmodified**, **modified**, or **staged**3种情况

### Git常用命令
#### `git add`
把working directory里标记为untraked的文件添加到stage

#### `git commit`
 recording a snapshot of your staging area
#### `git checkout`
#### `git status`

#### `git reset`

Git中3个树（树指文件的集合，不是数据结构中的树）HEAD, Index和工作区

HEAD指向上次commit的内容，Index指向将要被commit(即暂存区)的内容。 已经commit的内容和暂存区都位于.git文件夹，工作区相当于把.git内容解压出来。
图示：
{% asset_img git_three_tree.png %}
当运行`git reset`时发生了什么？
setp 1. 移动HEAD到指定的之前某一个commit
{% asset_img reset-soft.png %}
如果`reset`指令带`--soft`选项，`reset`指令执行到这一步停止。
这一步相当于撤销上次的commit，注意此时Index和工作区的内容没有变。

step 2. 更新暂存区
把HEAD指向的内容更新到暂存区， `reset`指令在默认情况下或者带`--mixed`选项，`reset`指令执行到这一步停止。这一步相当于撤销上次add操作。

setp 3. 更新工作区
把暂存区的内容更新到工作区中（执行到这一步需要`reset`指令带`--hard`选项）。在操作这一步时是危险的，因为会覆盖工作区的内容，之前工作区的文件不可恢复，需小心使用。

`reset`指令带文件或文件夹路径时：
指令会跳过步骤1（移动HEAD指向），因为HEAD是一个指针,指向commit的整体内容，不能指向部分内容，但是工作区和Index可以部分更新，所以跳过步骤1。
比如： `git reset file.txt`等于`git reset --mixed HEAD file.txt`。
等同于在命令行中运行`git status`，`Changes to be committed`标题下提示的这条命令`(use "git reset HEAD <file>..." to unstage)`