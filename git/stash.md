### 临时切换分支 管理暂存本地修改的代码
有多个开发分支时 进行本地分支切换时已修改代码的管理
例如：在dev分支进行开发时，想去release分支看一下代码逻辑。但是已经在dev分支做了代码，因业务没有完成，此时不能直接commit。至此，我们就需要用到git stash命令

具体流程如下：
```shell
git branch		// 查看当前分支
git stash			// 将本地改动暂存到“栈”界面
git checkout release		// 切换到release分支
```
这时再切回dev分支
```shell
git checkout dev		// 切换到dev分支
git stash show			//显示当前放在栈里的文件
git stash pop			// 再将刚存放在“栈”中的代码取出来
```