# Git

- 设置用户名和email

```shell
git config --global uesr.name "name"
git config --global user.email email
```

- 添加远程仓库

```shell
git remote add origin url
```

- 追踪文件

```shell
git add [name or dir]
```

- 提交到本地

```shell
git commit -m "message"
```

- 拉取

```shell
git pull origin master 
```

- 提交到远程仓库

```shell
git push -u origin master
```

- 版本回退

```shell
git log --oneline
git reset --hard [version_id]
```
