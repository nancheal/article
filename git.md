#GIT小结
>在使用git的过程中，学习到了很多，也遇到了很多的坑，记录下来

1.**在本地生成公钥和密钥**
```shell
ssh-keygen -t rsa -C "email@email.com"
```
2.**复制生成的key到github上**
```shell
cat /Users/pwd/.ssh/id_rsa.pub
```
3.**验证是否正常通讯**
```shell
ssh -T git@github.com
```
4.**设置username和email**
```shell
git config --global user.name "your name"
git config --global user.email "your_email@youremail.com"
```
5.**cd到要git的仓库**
```shell
git remote add origin git@github.com:yourName/yourRepo.git
git init
git add *.c
#git mv *.c file/ git 移动文件操作
git add README
git commit -m '初始化项目版本'
git push origin
```
**error情况**
1.出现有本地仓库和远程仓库commit不一的情况，慎用
```shell
git reset origin/master
```
这样会把本地仓库恢复成和远程一样的情况，可能会删除本地文件，坑啊,最好是本地先备份下
2.在删除之前的commit的时候我用了网上的下面的代码
```shell
git reset --hard <commit_id>
git push origin HEAD --force
```
也是一个会影响本地仓库的代码，慎用
3.出现fatal: Could not read from remote repository.
	可能是因为ssh验证和https验证的url格式的不同
    ```shell
    #https:
    https://github.com/xxxx/xxx.git
    #ssh
    git@github.com:xxxx/xxx.git
    ```