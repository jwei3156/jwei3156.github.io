---
title: "git基本使用"  
date: 2026-06-02T20:00:00+08:00
draft: false   
---

## Git

克隆项目

```
git clone https://github.com/jwei3156/test.git
```

使用ssh

```
git clone git@github.com:jwei3156/test.git
```

文件添加到暂存区

```
git add test.txt
```

暂存区的内容添加到本地仓库

```
git commit -m test.txt
```

进入仓库目录后，推送到github远程仓库同步

```
git push origin main
```

初始化仓库

```
git init
```

远程关联项目

```
git remote add origin github.com...
```



### 上传项目

设置上传人和邮箱

```
git config user.name ""
git config user.email ""
```

要上传的文件加入清单

```
git add -A
```

创建第一个本地版本

```
git commit -m "Initial commit"
```

创建github仓库，使用gh

```
gh repo create wexus --private --source=. --remote=origin
```

首次上传

```
git push -u origin main
```

浏览器打开查看

```
gh repo view --web
```



### 其他电脑同步

在开始修改前，查看是否有未提交的修改

```
git status
```

如果工作区干净，从github下载最新提交

```
git pull --ff-only
```

修改后，把修改加入待提交清单

```
git add -A
```

确认后，推送提交

```
git commit -m "描述这次修改"
```

最后，上传

```
git push
```

