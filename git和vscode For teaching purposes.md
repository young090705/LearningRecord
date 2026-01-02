## git使用

### Git的诞生

开源项目[linux kernel](https://zhida.zhihu.com/search?content_id=241003184&content_type=Article&match_order=1&q=linux+kernel&zhida_source=entity)开发，参与开发与维护者众多。1991至2005年期间绝大多数的 Linux 内核维护工作都花在了提交补丁和保存归档的繁琐事务上。 在2002 年，整个项目组开始启用一个专有的分布式版本控制系统 BitKeeper 来管理和维护代码，到了 2005 年，开发 BitKeeper  的商业公司同 Linux 内核开源社区的合作关系结束，他们收回了 Linux 内核社区免费使用 BitKeeper  的权力。

于是乎，Linus大神开始自己发力了，2005年4月[Linus Torvalds](https://zhida.zhihu.com/search?content_id=241003184&content_type=Article&match_order=1&q=Linus+Torvalds&zhida_source=entity) 开始为 Linux 内核开发一个新的版本控制系统，最初命名为 "[Git](https://zhida.zhihu.com/search?content_id=241003184&content_type=Article&match_order=1&q=Git&zhida_source=entity)"，这个名称是英文单词 "stupid" 的一个俚语，同年6月git的第一个原型发布。

2006 年3 月，Git 正式发布，并开始在 Linux 内核开发中使用。

### 检查git配置成功

#### 0.1检查git是否安装

```Plain
git --version
```

#### 0.2 创建用户名和用户邮箱

```Plain
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

#### **为什么要配置用户名和邮箱呢？**

因为Git是分布式版本控制系统，所以，每个机器都必须自报家门：你的名字和Email地址，这样远程仓库才知道哪次提交是==由谁完成==的。

### 使用Git的好处

- 可以帮忙管理版本，记录自己每天对这份代码改了什么内容，为什么去改这份代码
- 有时候误删内容，git帮你保存
- 可以和团队合作共同开发同一份代码的不同功能。

### Git前置知识

![Git常用命令 - 知乎](https://pic3.zhimg.com/v2-c86411661fe5ce883438783f641990ee_r.jpg)

上面四个圆柱体就是git底层的大致结构：工作区，暂存区，本地仓库，远程仓库

**工作区**：工作区就是我们写代码的区域，你在vscode修改文件，其实都发生在**工作区**。本质上他就是一个**文件夹**，在这个文件夹下我们初始化了git本地仓库，只是叫工作空间确实贴切点。

![image-20251030203742885](/home/rm_young/.config/Typora/typora-user-images/image-20251030203742885.png)

**暂存区**：暂存区的作用就是告诉git我要提交哪些文件。为什么要有暂存区的存在呢？如果没有了暂存区，我们直接使用commit就会一股脑提交所有更改了的文件和修改的文件，但我有时候可能想分多次提交，比如，`main.cpp` 我就修了个bug,那我就单独提交一个’fix bug‘，`utils.cpp`我把算法从O（N^2）优化到O（N），那我就提交一个"optimize algorithm"。最后再补充个readme.md的说明文档。

**本地仓库** ：本地仓库就是Git 把暂存区的内容，正式写入本地仓库，生成一个新的 **提交（commit）对象**。

**远程仓库**：有点像steam，就是保存游戏进度，在别的电脑，只要有steam账号，就可以加载游戏进度。github就是这样，只不过保存的是代码进度，只要你有你的github账号密码，随时可以加载你的代码，并进行修改。



### 图形化界面的git

1.创建文件夹，然后在vscode点击初始化仓库即可

==往后的东西由于时间原因，下去有空的可以自己去看看==

2.给此次提交命名，然后提交

3.如果想直接用**文件夹名称**作为项目名称,那直接发布就好了，但如果想要自己再创建一个名称的话，可以自己现在github上创建一个项目名称，然后按一下步骤添加远程仓库
![](/home/rm_young/图片/截图/截图 2025-07-24 20-31-47.png)

然后顶部自己就会出现添加远程仓库。按照步骤做就行了。

![image-20250724212608931](/home/rm_young/.config/Typora/typora-user-images/image-20250724212608931.png)

这里可以添加`github`为名称就可以了。这个是当你一个本地仓库对应多个远程仓库的时候，要用的,推荐用对应平台的名称就好了。

完成上述操作后，就完成了==初始化本地仓库和链接远程仓库==。

### 常用指令

1. 创建版本

```Python
#提交到暂存区
git add 文件名称
git add . #添加所有未跟踪的文件或者所有更改了的文件
#提交到本地仓库
git commit -m 'Version one'
```

2. 查看版本记录

```Python
git log
git log --pretty=online#简化输出的版本
```

3. 回退版本

- HEAD后面添加多少^就是回退多少个版本，但如果要回退100个版本呢？就可以用~+X来回退x个版本

```Python
git reset --hard HEAD^
git reset --haed HEAD~100 #回退100个版本
```

- 也可以利用git log来了解版本号，然后设置版本

```Python
git log
git reset --hard 版本号（这个版本号可以用之前的gitlog读取出来）
```

终端关了，没法用gitlog找序列号怎么办

可以用下面这句命令来查阅操作记录

```Python
git reflog
```

查阅出来，最前面的数字序列就是我们要的版本号；

4. 删除的是我们在**工作区**的文件

```c
rm 文件名
git rm 文件名 //把删除的改动添加到暂存区
git commit -m '版本名称' //版本名称是自主命名的
```

这个删除文件==本质也是创建版本==，在上面的各个阶段，我们也可以合理运用上**撤销修改**,比如运行完指令`rm 文件名`,后悔了，可以用`git checkout -- 文件名`,再比如**已经把东西提交到仓库了**，但我们后悔文件删除了，我们就可以使用版本回退。

### git分支

![image-20250725141525756](/home/rm_young/.config/Typora/typora-user-images/image-20250725141525756.png)

创建并切换分支如上效果，然后我们就可以在dev分支独立开发，**直到后面完成开发**后，开始合并分支HEAD重新指向`master`。

以下的name统一指的是**分支名称**如dev这样的，自主命名的

```python
git branch#查看当前有几个分支
git checkout -b <name> #创建并且切换分支
git branch <name> #创建分支
git checkout <name> #切换分支
git merge <name> #合并分支
git branch -d <name> #删除分支
```

**合并分支并不是一帆风顺的**，这时候就需要**解决冲突**

那什么时候会起冲突呢？当你在两个分支都进行了提交，**且修改的是同一个文件**，此时在合并分支时就会引发冲突。

怎么解决冲突呢？答案是**手动对冲突文件进行修改**

![image-20250819103202910](/home/rm_young/.config/Typora/typora-user-images/image-20250819103202910.png)

这里把第6行和第8行和第10行删掉就可以了。

### gitignore的使用

[csdn大佬教学](https://blog.csdn.net/m0_63230155/article/details/134471033)











### 总结

去用Git

