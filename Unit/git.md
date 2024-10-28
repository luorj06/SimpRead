# git常用命令


| 命令 | 解析 |
| ---- | ---- |
| git branch | 显示本地分支 |
| git branch --contains 50089 | 显示包含提交 50089 的分支 |
| git branch --merged | 显示所有已合并到当前分支的分支 |
| git branch --no-merged | 显示所有未合并到当前分支的分支 |
| git branch -a | 显示所有分支 |
|git branch -d hotfixes/BJVEP933 | 删除分支 hotfixes/ |
| git branch -D hotfixes/BJVEP933 | 强制删除分支 hotfixes/BJVEP933 |
| git branch -m master master_copy | 本地分支改名 |
| git branch -r | 显示所有原创分支 |
| git checkout -- README | 检出 head 版本的 README 文件（可用于修改错误回退） |
| git checkout --track hotfixes/BJVEP933 | 检出远程分支 hotfixes/BJVEP933  |
| git checkout -b devel origin/develop | 从远程分支 develop 创建新本地分支  |
| git checkout -b master master_copy | 上面的完整版 |
| git checkout -b master_copy | 从当前分支创建新分支 master_copy 并检出 |
| git checkout features/performance | 检出已存在的 features/performance 分支 |
| git checkout v2.0 | 检出版本 v2.0 |
| git cherry-pick ff44785404a8e | 合并提交 ff44785404a8e 的修改 |
| git diff | 显示所有未添加至 index 的变更 |
| git diff --cached | 显示所有已添加 index 但还未 commit 的变更 |
| git diff HEAD -- ./lib | 比较与 HEAD 版本 lib 目录的差异 |
| git diff HEAD^ | 比较与上一个版本的差异 |
| git diff origin/master..master | 比较远程分支 master 上有本地分支 master  |
| git diff origin/master..master --stat | 只显示差异的文件，不显示具体内容 |
| git fetch | 获取所有远程分支（不更新本地分支，另需 merge） |
| git fetch --prune | 获取所有原创分支并清除服务器上已删掉的分支 |
| git grep "delete from" | 文件中搜索文本 “delete from” |
| git log | 显示提交日志 |
| git log --pretty=format:'%h %s' --graph | 图示提交日志 |
| git log --stat | 显示提交日志及相关变动文件 |
| git log v2.0 | 显示 v2.0 的日志 |
| git ls-files | 列出 git index 包含的文件 |
| git ls-tree HEAD | 内部命令：显示某个 git 对象 |
| git merge origin/master | 合并远程 master 分支至当前分支 |
| git reflog | 显示所有提交，包括孤立节点 |
| git remote add origin git+ssh://git@192.168.53.168/ | VT.git |
| git reset --hard HEAD | 将当前版本重置为 HEAD（通常用于 merge 失败回退） |
|git revert dfb02e6e4f2f7b573337763e5c0013802e392818 | 撤销提交  |
| git show HEAD | 显示 HEAD 提交日志 |
| git show HEAD^ | 显示 HEAD 的父（上一个版本）的提交日志 为上两个版本 5 为上 5  |
| git show master@{yesterday} | 显示 master 分支昨天的状态 |
| git show v2.0 | 显示 v2.0 的日志及详细内容 |
| git show-branch | 图示当前分支历史 |
| git stash | 暂存当前修改，将所有至为 HEAD 状态 |
| git stash apply stash@{0} | 应用第一次暂存 |
| git stash list | 查看所有暂存 |
| git stash show -p stash@{0} | 参考第一次暂存 |
| git push --tags | 把所有 tag 推送到远程仓库 |
| git tag | 显示已存在的 tag |
| git tag -a v2.0 -m 'xxx' | 增加 v2.0 的 tag |
| git whatchanged | 显示提交历史对应的文件修改 |