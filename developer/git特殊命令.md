# git上的特殊命令

* 统计每个人的提交量
```
git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
```


* check单个文件

相当方便, 只要代码写的好一点, 功能独立就十分好迁移了.

```
git checkout branch-name path
```

* 查看合并分支的日志

可以在对合并后的分支进行单个分支提交记录查询.

```
# 显示branch1中排除branch2的日志记录
git log branch1 ^branch2
```
