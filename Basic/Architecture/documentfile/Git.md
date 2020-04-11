# Git

## 1. git简介

1. 1991年,linus创建了开源的Linux，从此，Linux系统不断发展，已经成为最大的服务器系统软件了,Linux的壮大是靠全世界热心的志愿者参与的，这么多人在世界各地为Linux编写代码,2002年以前，世界各地的志愿者把源代码文件通过diff的方式发给Linus，然后由Linus本人通过手工方式合并代码！
2. 2002年，Linux系统已经发展了十年了，代码库之大让Linus很难继续通过手工方式管理了，社区的弟兄们也对这种方式表达了强烈不满，于是Linus选择了一个商业的版本控制系统BitKeeper，BitKeeper的东家BitMover公司出于人道主义精神，授权Linux社区免费使用这个版本控制系统。
3. 2005年,Linux社区牛人聚集，不免沾染了一些梁山好汉的江湖习气。开发Samba的Andrew试图破解BitKeeper的协议（这么干的其实也不只他一个），被BitMover公司发现了（监控工作做得不错！），于是BitMover公司怒了，要收回Linux社区的免费使用权。
4. 2005年，Linux内核开发团队与BitMover公司进行磋商，但无法解决他们之间的歧见。林纳斯·托瓦兹（林纳斯·掏袜子）决定自行开发版本控制系统替代BitKeeper，以十天的时间用C编写出git第一个版本。Git迅速成为最流行的分布式版本控制系统。
5. 2008年，GitHub网站上线了，它为开源项目免费提供Git存储，无数开源项目开始迁移至GitHub，包括jQuery，PHP，Ruby等等。

### 1.1 命名来源

- [git](https://zh.wiktionary.org/wiki/en:git)”，该词源自英国俚语，意思大约是“混账”。[林纳斯·托瓦兹](https://zh.wikipedia.org/wiki/林纳斯·托瓦兹)自嘲地取了这个名字。

- I'm an egotistical bastard, and I name all my projects after myself. First [Linux](https://zh.wikipedia.org/wiki/Linux), now git.

### 1.2 分布式控制系统

- 控制系统分，集中式和分布式控制系统。git使用的分布式控制系统。

1. 集中式控制系统，数据存放在中央服务器，增，删文件都是在中央服务器取得最新版本的文件，在本地联网，然后对其修改，修改完以后在把文件推送给中央服务器。集中式控制系统最大的问题是必须联网才能修改，网速不好提交一个大型文件会让人等到崩溃。要是中央服务器出问题了，所有人都不能登录修改文件。
2. 分布式控制系统，数据保存在自己本地，提升性能和并发，操作被分发到不同的，相互独立，提升系统的可用性，即使部分分片不能用，其他分片不会受到影响。

## 2. 安装git

- 最早Git是在Linux上开发的，很长一段时间内，Git也只能在Linux和Unix系统上跑。不过，慢慢地有人把它移植到了Windows上。现在，Git可以在Linux、Unix、Mac和Windows这几大平台上正常运行了。

- 官网：<https://git-scm.com/downloads>

### 2.1 linux上安装

```bash
#安装
yum install git
#认为版本太旧，可以使用下面命令更新
git clone https://github.com/git/git
```

### 2.2 windows 上安装git

- 直接去官网下载，64位的版本，安装。
- 安装直接下一步下一步即可。（傻瓜式操作）
- 下载完后，点开菜单，选择Git --Git Bash 或者在主屏幕点击右键选择Git Bash Here。



- 选择Git GUI Here是window系统下操作，Git Bash Here 是基于windows系统进行linux操作。

## 3.Git命令

```bash
#查看你版本
$ git --version
git version 2.21.0.windows.1

#全局认证 --global
$ git config --global user.name "Your Name" 
$ git config --global user.email "email@example.com"
```

### 3.1 仓库：repository

1. 仓库，可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改、删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻可以“还原”。

```bash
#创建库，在window中创建库，库名的父进程不要是中文，不然会出现各种意想不到的错误。
$ mkdir learngit
$ cd learngit
$ pwd
/Users/michael/learngit
```

2. 管理仓库

```bash
#将创建的目录变成管理的仓库
$ git init
Initialized empty Git repository in /Users/michael/learngit/.git/

#还有一个隐藏目录，是用来跟着管理版本库的
ls -ah

```

### 3.2 上传文件



```bash
使用git add filename将filename文件添加到index(暂存区)中
使用git commit filename将filename文件从index(暂存区)添加到创库(Repository)中。
使用git push 将本地仓库(Repository)添加到远程仓库Remote中
git config --global user.name "name"	配置全局的用户名信息为name
git config --global user.email "email"	配置全局的用户邮箱为email
git config --global user.name	显示user.name信息
git config --global user.email	显示user.email信息
注意：上面命令配置的信息会存放在当前用户家目录下面~/.gitconfig文件中记录


cat ~/.gitconfig

```



```bash



#用命令行上创建一个新的存储库： create a new repository on the command line

echo "# linux.github.io" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin git@github.com:Lqlixl/linux.github.io.git
git push -u origin master


#从命令行推送现有存储库：push an existing repository from the command line


git remote add origin git@github.com:Lqlixl/linux.github.io.git
git push -u origin master



#从另一个存储库导入代码：import code from another repository



git config --global user.name "Administrator"
git config --global user.email "admin@example.com"

Create a new repository
git clone http://www.lqlixl.com/study/web1.git
cd web1
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

Push an existing folder
cd existing_folder
git init
git remote add origin http://www.lqlixl.com/study/web1.git
git add .
git commit -m "Initial commit"
git push -u origin master

Push an existing Git repository
cd existing_repo
git remote rename origin old-origin
git remote add origin http://www.lqlixl.com/study/web1.git
git push -u origin --all
git push -u origin --tags



ssh-keygen -t rsa -C "youremail@example.com"
-f 指定路径
文件必须存在
```



### 4.版本回滚

```bash
git log
git reflog
git reset --hard HEAD^ 


```

### 4.1小结

```bash

要重返未来，用git reflog查看命令历史，以便确定要回到未来的哪个版本。
要随时掌握工作区的状态，使用git status命令。
如果git status告诉你有文件被修改过，用git diff可以查看修改内容

HEAD指向的版本就是当前版本，因此，Git允许我们在版本的历史之间穿梭，使用命令git reset --hard commit_id。
穿梭前，用git log可以查看提交历史，以便确定要回退到哪个版本。

场景1：当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令git checkout -- file。
场景2：当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令git reset HEAD <file>，就回到了场景1，第二步按场景1操作。
场景3：已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库。

命令git rm用于删除一个文件。如果一个文件已经被提交到版本库，那么你永远不用担心误删，但是要小心，你只能恢复文件到最新版本，你会丢失最近一次提交后你修改的内容。
git rm  
git commit
git checkout -- test.txt  相当于还原删除的
```







### 5.创建分支

```bash
#在web界面可以添加分支

#查看当前分支
 git branch

git checkout -b dev  #git checkout命令加上-b参数表示创建并切换，相当于以下两条命令：
$ git branch dev  # 创建dev分支
$ git checkout dev  # 切换到dev分支

#在dev分支修改文件，然后在提交add  commit    ;然后回到master分支
合并两个分支
git merge dev #将dev分支合并到当前分支
git branch -d dev  #删除dev分支

git switch + 分支  #切换分支
git switch  -c 分支  #创建并切换分支
```

#### 5.1master和dev分支同时修改

```bash
git checkout -b dev   #创建分支切换到分支，并对分支进行修改，然后add commit ；
git checkout master   #切换回master,并对master分支进行修改，然后add  commit;
git merge dev         #合并dev分支到master分支，就会报错，需要手动解决
git status            #使用status查看
#查看修改的文件，Git用<<<<<<<，=======，>>>>>>>标记出不同分支的内容，我们修改如下后保存；
# 修改后在此提交，add+commit
git log --graph --pretty=oneline --abbrev-commit #使用git log查看过程
#用git log --graph命令可以看到分支合并图。


#git 的 merge 与 no-ff merge 的不同之处\
通常，合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。
#实例
新建分支dev1，修改readme.txt，然后在dev1分支下git add readme.txt  git commit -m "dev1 branch commit"
回到master分支，执行merge即git merge dev1
删除分支
查看日志即git log --graph --pretty=oneline --abbrev-commit
新建分支dev2，修改readme.txt，然后在dev2分支下git add readme.txt  git commit -m "dev2 branch commit"
回到master分支，执行merge即git merge --no-ff -m "dev2 merged with mo-ff" dev2
删除分支
查看日志即git log --graph --pretty=oneline --abbrev-commit
比较两次合并，可以看出不同之处，no-ff的模式会记录分支历史、


```

### 5.2 master出问题了，dev分支也发生修改

```bash
#将dev分支隐藏起来
git stash  #隐藏
git stash list   #查看隐藏的分支
#恢复隐藏工作的分支
1.用git stash apply恢复，但是恢复后，stash内容并不删除，你需要用git stash drop来删除；
2.git stash pop    恢复的同时把stash内容也删了

可以多次stash，恢复的时候，先用git stash list查看，然后恢复指定的stash，用命令：
$ git stash apply stash@{0}

# 在master分支上修复的bug，想要合并到当前dev分支，可以用git cherry-pick <commit>命令，在commit的时候有个编号（4c805e2），把bug提交的修改“复制”到当前分支，避免重复劳动。

```

#### 5.3 开发新的主题

```bash
开发一个新feature，最好新建一个分支；
如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name>强行删除
```



### 6.多人协作

```bash
从远程仓库克隆时，实际上Git自动把本地的master分支和远程的master分支对应起来了，并且，远程仓库的默认名称是origin。要查看远程库的信息
#git remote
#git remote -v 详细信息
抓取和推送的origin的地址。如果没有推送权限，就看不到push的地址。


```

#### 6.1推送分支

```bash
#推送分支，就是把该分支上的所有本地提交推送到远程库。推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上：
$ git push origin master

#如果要推送其他分支，比如dev，就改成：
$ git push origin dev

#but，并不是一定要把本地分支往远程推送，那么，哪些分支需要推送，哪些不需要呢？
master分支是主分支，因此要时刻与远程同步；
dev分支是开发分支，团队所有成员都需要在上面工作，所以也需要与远程同步；
bug分支只用于在本地修复bug，就没必要推到远程了，除非老板要看看你每周到底修复了几个bug；
feature分支是否推到远程，取决于你是否和你的小伙伴合作在上面开发。
总之，就是在Git中，分支完全可以在本地自己藏着玩，是否推送，视你的心情而定！

```

#### 6.2 other合作者

```bash
#先clone
要在dev分支上开发，就必须创建远程origin的dev分支到本地，于是他用这个命令创建本地dev分支
# git checkout -b dev origin/dev
现在就可以在dev上继续修改，然后，时不时地把dev分支push到远程：
# git push origin dev

小伙伴已经向origin/dev分支推送了他的提交，而碰巧你也对同样的文件作了修改，并试图推送会失败，需要先
# git pull
git pull也失败了，原因是没有指定本地dev分支与远程origin/dev分支的链接，根据提示，设置dev和origin/dev的链接：
# git branch --set-upstream-to=origin/dev dev
然后在次 pull
然后在次push
```

#### 6.3 总结

```bash
多人协作的工作模式通常是这样：
	首先，可以试图用git push origin <branch-name>推送自己的修改；
	如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
	如果合并有冲突，则解决冲突，并在本地提交；
	没有冲突或者解决掉冲突后，再用git push origin <branch-name>推送就能成功！
	如果git pull提示no tracking information，则说明本地分支和远程分支的链接关系没有创建，用命令git branch --set-upstream-to <branch-name> origin/<branch-name>。
	这就是多人协作的工作模式，一旦熟悉了，就非常简单。
小结
	查看远程库信息，使用git remote -v；
	本地新建的分支如果不推送到远程，对其他人就是不可见的；
	从本地推送分支，使用git push origin branch-name，如果推送失败，先用git pull抓取远程的新提交；
	在本地创建和远程分支对应的分支，使用git checkout -b branch-name origin/branch-name，本地和远程分支的名称最好一致；
	建立本地分支和远程分支的关联，使用git branch --set-upstream branch-name origin/branch-name；
	从远程抓取分支，使用git pull，如果有冲突，要先处理冲突。
	
	#查看分支
	git log --graph --pretty=oneline --abbrev-commit
	#rebase 变基（垫底）整理
	原本分叉的提交现在变成一条直线
	git rebase
	rebase操作可以把本地未push的分叉提交历史整理成直线；
    rebase的目的是使得我们在查看历史提交的变化时更容易，因为分叉的提交需要三方对比。
```

### 7.tag号

```bash
commit号:6a5819e
tag：版本号是v1.2”
通过tagv1.2查找commit就行！”
tag就是一个让人容易记住的有意义的名字，它跟某个commit绑在一起。
标签虽然是版本库的快照，但其实它就是指向某个commit的指针
tag是打在最新提交的commit上
```

### 7.1创建tag号

```bash
#切换到要打标签的分支
 git branch
git checkout master
#打标签tag号
git tag v1.0
#查看所有标签
git tag

#上周的commit忘记打tag
找到历史提交的commit id，然后打上就可以了
git log --pretty=oneline --abbrev-commit
#找到commit号：f52c633
git tag v0.9 f52c633

#标签不是按时间顺序列出，而是按字母排序的。可以用git show <tagname>查看标签信息：
git show v0.9

#还可以创建带有说明的标签，用-a指定标签名，-m指定说明文字：
git tag -a v0.1 -m "version 0.1 released" 1094adb
用命令git show <tagname>可以看到说明文字
不加-a的话，show的时候会发现没有这行：“tag:v0.1”

# 注意：标签总是和某个commit挂钩。如果这个commit既出现在master分支，又出现在dev分支，那么在这两个分支上都可以看到这个标签
```

#### 7.3 操作标签

```bash
#标签打错了，也可以删除：
git tag -d v0.1

#推送某个标签到远程
创建的标签都只存储在本地，不会自动推送到远程。所以，打错的标签可以在本地安全删除。如果要推送某个标签到远程，使用命令git push origin <tagname>：
git push origin v1.0
#一次性推送全部尚未推送到远程的本地标签
git push origin --tags
#如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除：
git tag -d v0.9
#然后，从远程删除。删除命令也是push，但是格式如下
git push origin :refs/tags/v0.9
```

### 7.2小结

```bash
小结
命令git tag <tagname>用于新建一个标签，默认为HEAD，也可以指定一个commit id；
命令git tag -a <tagname> -m "blablabla..."可以指定标签信息；
命令git tag可以查看所有标签。

命令git push origin <tagname>可以推送一个本地标签；
命令git push origin --tags可以推送全部未推送过的本地标签；
命令git tag -d <tagname>可以删除一个本地标签；
命令git push origin :refs/tags/<tagname>可以删除一个远程标签。
```

```bash

在GitHub上，可以任意Fork开源仓库；

自己拥有Fork后的仓库的读写权限；

可以推送pull request给官方仓库来贡献代码
```





### 8.上传错误

- 上传到仓库，发现有灰色图标

```bash
git rm --cached <folder_name>git add .
git commit -m "<your_message>"
git push --all
```

