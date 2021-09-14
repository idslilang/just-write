### 1.重置历史提交记录

 

```
基于当前分支创建一个独立的分支
git checkout --orphan  dev_temp

添加所有文件变化至暂存空间
git add .

提交并添加提交记录
git commit -m "commit message"

删除当前分支
git branch -D master

重新命名当前独立分支为 master
git branch -m master

推送到远端分支

git push -f origin master


```



