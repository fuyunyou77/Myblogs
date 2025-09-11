---
title: Git操作笔记
tags:
  - git
excerpt: git的常用操作介绍
categories:
  - 工具
  - git
date: 2025-09-11 16:06:21
---
## Git 基本结构

![image-20240719170942940](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/image-20240719170942940.png)

> 图示说明了 Git 工作区、暂存区、本地仓库和远程仓库之间的关系

## Git 配置

### 查看配置

```
git config --list
```

### 配置用户信息

​**全局配置**​（对所有代码库生效）：

```
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

​**局部配置**​（只对当前代码库有效）：

```
git config --local user.name "你的名字"
git config --local user.email "你的邮箱"
```

配置后，远程仓库提交的 commit 里对应的用户即为 `user.name`。

### 配置代理

​**删除错误的代理配置**​：

```
git config --global --unset https.proxy
```

​**设置正确的代理**​：

```
git config --global http.proxy http://127.0.0.1:10810
git config --global https.proxy http://127.0.0.1:10810
```

> [!info] 代理端口号说明
> 
> 这里的端口号是本地代理端口号，需根据自己使用的代理工具查看。
> 
> 以 v2rayn 为例，查看"本地混合监听端口"参数，一般为 10810 或 10808。

​**验证配置**​：

```
git config --global --get http.proxy
git config --global --get https.proxy
```

## Git 基本操作

### 创建版本库

​**初始化本地仓库**​：

```
git init
```

​**克隆远程仓库**​：

```
git clone <url>
```

​**克隆特定分支**​：

```
git clone -b branch_name url
# 示例
git clone -b Branch_1GE_XPON_BRANCH_WAN_INOTECH git@192.168.2.241:/home/data/git/ONU/zte9125/zte9125_prj.git
```

### 修改与提交

|命令|说明|
|---|---|
|`git add <file>`|将单个文件从工作区添加到暂存区|
|`git add .`|将所有修改的文件添加到暂存区|
|`git commit -m "message"`|将暂存区文件提交到本地仓库|
|`git status`|查看工作区状态，显示有变更的文件|
|`git diff`|比较文件差异（暂存区和工作区）|

​**合并到上次提交**​（适用于补充上次提交遗漏的内容）：

```
git commit --amend [--no-edit]  # 加上 --no-edit 不修改提交信息
```

### 远程仓库操作

|命令|说明|
|---|---|
|`git push origin master`|将本地 master 分支推送到远程|
|`git pull`|下载远程代码并合并（= `git fetch`+ `git merge`）|
|`git fetch`|从远程获取代码库，但不合并|
|`git remote add origin <url>`|关联远程仓库|
|`git remote -v`|查看远程库信息|

## 撤销与回退操作

### 撤销操作

​**场景 1**​：改乱了工作区文件，尚未添加到暂存区

```
git checkout <file>  # 撤销工作区文件到暂存区状态
```

​**场景 2**​：改乱了工作区文件并已添加到暂存区

```
git reset HEAD <file>    # 第一步：撤销暂存区的修改
git checkout <file>      # 第二步：撤销工作区的修改
```

​**场景 3**​：乱改了很多文件，想回到最新一次提交状态

```
git reset --hard HEAD  # 撤销所有未提交文件的修改
```

### 回退操作（已提交）

|命令|说明|
|---|---|
|`git reset --hard HEAD^`|回退到上次提交|
|`git reset --hard HEAD~n`|回退到 n 个版本前|
|`git reset --hard <commitid>`|回退到指定 commit|
|`git reset --soft <commitid>`|回退到指定 commit，保留暂存区内容|
|`git reset --mixed <commitid>`|回退到指定 commit，保留工作区内容（默认）|

## Git 分支管理

### 基本分支操作

|命令|说明|
|---|---|
|`git branch <name>`|创建分支|
|`git checkout <name>`|切换到分支|
|`git checkout -b <name>`|创建并切换到新分支|
|`git merge <name>`|合并分支到当前分支|
|`git branch -a`|查看所有分支（本地和远程）|
|`git branch -d <name>`|删除分支|

### 临时存储更改

```
git stash          # 暂时存储当前更改
git stash pop      # 恢复暂存的更改
```

> 适用于：当前工作未完成但需要临时切换分支的情况。

### 分离头指针状态处理

​**检查状态**​：

```
git status
# 输出：HEAD detached from ed71a86
```

​**恢复"丢失"的提交**​：

```
git reflog                      # 查找最近的分离头指针提交
git branch recovered-commit abc1234  # 创建分支指向该提交
git switch main                # 切换回主分支
git merge recovered-commit     # 合并恢复的提交
```

### 分支查看与删除

​**查看分支**​：

```
git branch      # 查看本地分支
git branch -r   # 查看远程分支
git branch -a   # 查看所有分支
```

​**删除分支**​：

```
# 删除本地分支
git branch -d <branch_name>    # 安全删除（检查合并状态）
git branch -D <branch_name>    # 强制删除

# 删除远程分支
git push origin --delete <branch_name>
# 或
git push origin :<branch_name>
```

## 代码回退策略

### 1. reset 回退（彻底回退到某一版本）

```
git log                 # 查看提交历史
git reset --hard <目标版本号>  # 回退到指定版本
git push -f             # 强制推送到远程仓库
```

### 2. revert 回退（安全回退，保留历史）

```
git log                 # 查看提交历史
git revert <要回退的版本号>  # 回退该版本代码并生成新版本
git status              # 查看变化文件
git add .               # 添加到暂存区
git commit -m "回退说明"   # 提交到本地仓库
git push                # 推送到远程分支
```

> ​**区别**​：
> 
> - `reset`：彻底回退，之后版本的代码都不保留
>     
> - `revert`：只回退指定版本的更改，其他版本不受影响，更安全
>     

## 文件操作

### 删除文件

```
# 删除单个文件
rm -f 文件名

# 批量删除文件
rm -f *关键字*

# 删除文件夹
rm -rf 文件夹名
```

## 其他常用命令

### 查看提交历史

```
git log --oneline          # 简洁显示
git log --graph           # 图形化显示分支结构
git log --stat            # 显示文件修改统计
git log -p               # 显示具体修改内容
```

### 标签管理

```
git tag                  # 查看所有标签
git tag v1.0.0          # 创建轻量标签
git tag -a v1.0.0 -m "版本发布说明"  # 创建附注标签
git push origin --tags   # 推送所有标签到远程
```

### 忽略文件配置

创建 `.gitignore`文件来指定需要忽略的文件和文件夹：

```
# 忽略日志文件
*.log

# 忽略编译生成文件
/build/
/dist/

# 忽略系统文件
.DS_Store

# 忽略IDE配置
.idea/
*.swp
*.swo
```

### 比较差异

```
git diff --staged       # 比较暂存区与最新提交
git diff HEAD          # 比较工作区与最新提交
git diff branch1..branch2  # 比较两个分支
```

### 重写历史（谨慎使用）

```
git rebase -i HEAD~3   # 交互式变基，修改最近3个提交
```

> ​**注意**​： rebase 会重写提交历史，在团队协作中需谨慎使用。

---

## 参考资料

1. [Git 教程 - 菜鸟教程](https://www.runoob.com/git/git-server.html)
    
2. [Git 代码回退详解](https://blog.csdn.net/qing040513/article/details/109150075)
    
3. [Git 分支管理实践](https://zhuanlan.zhihu.com/p/404642045)
    

> 本笔记持续更新，建议定期回顾和补充实际使用中的经验总结。