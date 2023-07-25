##### git:本地库与远程库建立连接

要使用Git上传文件到GitHub上新建的仓库，可以按照以下步骤进行操作：

1. 在本地计算机上安装Git，并确保已正确配置Git的用户名和电子邮件地址。
2. 在GitHub上创建一个新的仓库，获取仓库的URL。
3. 在本地计算机上打开终端或命令行界面，进入要上传文件的项目目录。
4. 初始化Git仓库：

```shell
git init
```

5. 将文件添加到Git仓库中：

```shell
git add <文件名>
```

6. 如果要将整个目录添加到仓库中，可以使用以下命令：

```shell
git add .
```

7. 提交文件到本地仓库：

```shell
git commit -m "提交说明"
```

在引号中替换为你的提交说明，例如 "添加新文件"。

8. 关联本地仓库和GitHub仓库：

```shell
git remote add origin <GitHub仓库URL>
```

将`<GitHub仓库URL>`替换为你在第2步获取到的仓库URL。

9. 推送本地仓库的内容到GitHub仓库：

```shell
git push -u origin master
```



这会将文件推送到名为`master`的分支上。如果使用的是其他分支，将`master`替换为相应的分支名称。

完成以上步骤后，Git会将文件上传到GitHub仓库中。可以在GitHub上查看仓库，确认文件是否成功上传。

