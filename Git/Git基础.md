### 工作流程
<img src="Pasted image 20260308194827.png" alt="alt text" width="700">

### 基础命令

- `git  add   文件名.后缀`  工作区 -> 暂存区
- `git commit   -m"注释内容，会在日志中显示"`        暂存区 -> 本地仓库
- `git log [option]` 查看日志
- options<br>
	- --all 显示所有分支
	- --pretty=oneline 将提交信息显示为一行
	- --abbrev-commit 使得输出的commitId更加简短
	- --graph 以图的形式显示

- `git status`  查看修改状态
- `git reset--hard commitID` 版本回退 
- `git reflog`  查看已经删除的提交记录
- 忽略文件,创建一个gitingnore文件，`touch .gitignore`，在文件中列出要忽略的文件模式
### Git分支
>把工作从开发主线上分离开来进行，例如bug修改、开发新功能等，以免影响主线开发

- `git branch` 查看本地分支
- `git branch 分支名`     创建一个分支
- `git checkout  分支名`  切换到指定分支
- `git checkout -b 分支名`  创建并切换到新分支
- `git merge 分支名`  合并分支
- `git branch -d`    删除分支
  -D   强制删除
![[Pasted image 20260308195510.png]]
### 远程仓库
- 设置SSH公钥到当前文件夹，文件名为id_rsa  
>`ssh-keygen -t rsa -f./id_rsa`   不断回车 公钥会自动覆盖已经存在的
- 获取公钥  `cat  ./id_rsa.pub`
![[Pasted image 20260308195748.png]]
- 克隆远程仓库 `git clone <仓库路径>[本地目录]`