# git常用命令

1. 上传修改的文件到暂存区（将未跟踪文件状态 变为 Staged状态）`git add .`
2. 将修改同步到本地仓库 `git commit -m "消息内容"`  
3. `git push [origin] [分支名]` 将修改的内容 提交到远程仓库(也可以省略 origin 分支名)
4. `git init` 在本地创建一个版本控制的文件夹
5. `git remote add 仓库别名 远程仓库地址` 添加远程仓库
6. `git pull [仓库别名] [分支名]` (也可以不加 直接全部pull进来)
7. `git clone 远程仓库地址`  克隆一份下来 clone后就自动添加这个远程仓库了
8. `git status` 查看状态
9. 设置用户名和邮箱 ===>安装git 后第一件事

```bash
git config --global user.name "xxxxx" 
git config --global user.emainl cect3329@163.com
#--global 为全局配置   不设置--global的话，则可以在一个特定的项目中使用不同的名称或者e-mail地址
```

10. 查看不同级别的配置文件：

配置文件查看：系统配置文件===>git安装目录下的gitconfig  Git\etc\gitconfig

当前用户配置文件 ===>在User文件夹下的.gitconfig

```bash
#查看系统config
git config --system --list
#查看当前用户的config配置
git config --global --list
```

11. 设置本机绑定的SSH公钥，实现免密码登录

```bash
ssh-keygen -t rsa
#生成两个文件 在用户文件夹下的.ssh文件夹中，id_rsa.pub 为公钥 id_rsa好像是加密的
```



