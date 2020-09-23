---
title: git 多账户配置
date: 2020-07-19 11:34:12
tags: [git]

---
参考:

[https://www.cnblogs.com/hanguozhi/p/10878043.html]()
[https://juejin.im/post/5d1ebab8f265da1bd2610b6b]()

- 1，之前配置过全局的用户名的先删掉，
查看全局用户名和邮箱的命令：

```
git config --global user.name
git config --global user.email
```

删除全局用户名和邮箱

```
git config --global --unset user.name 
git config --global --unset user.email
```

2，先生成码云的私钥和公钥
进入目录

```
cd ~/.ssh
ssh-keygen -t rsa -C “2259434901@qq.com”
```

会提示你输入公钥和私钥的命名

```
Generating public/private rsa key pair. 
Enter file in which to save the key (/Users/liugui/.ssh/id_rsa):
输入 gitee_rsa
```

3， 私钥添加到本地

```
ssh-add ~/.ssh/gitee_rsa
```

可以用 ssh-add -l 查看是否添加成功

4，公钥上传到码云网址

5， 验证

```
ssh -T git@gitee.com 
```

6， 再生成github 的私钥和公钥,步骤和之前差不多，要区别名字
进入目录

```
cd ~/.ssh

ssh-keygen -t rsa -C “1179169386@qq.com”
```

会提示你输入公钥和私钥的命名

```
Generating public/private rsa key pair. 
Enter file in which to save the key (/Users/liugui/.ssh/id_rsa):

输入 github_rsa
```

7,  私钥添加到本地

```
ssh-add ~/.ssh/github_rsa
```

可以用 `ssh-add -l` 查看是否添加成功

8，公钥上传到码云网址

9 ，验证

```
ssh -T git@github.com 
```

10，进入目录，新建配置文件

```
cd ~/.ssh
touch config
```
config 文件里内容：

```
Host gitee
HostName gitee.com
 PreferredAuthentications publickey
 User 2259434901@qq.com
 IdentityFile ~/.ssh/gitee_rsa

#ssh -T GitHub 会报错， ssh -T git@github.com 不报错
Host github
 HostName github.com
 PreferredAuthentications publickey
 User 1179169386@qq.com
 IdentityFile ~/.ssh/github_rsa

# ssh -T zhihan 会报错， ssh -T git@zhihanyun.com 不报错
Host zhihan
 HostName zhihanyun.com
 PreferredAuthentications publickey
 User 2259434901@qq.com
 IdentityFile ~/.ssh/gitlab_rsa
```

11， 奇怪的是github 用别名验证会失败

```
ssh -T gitee //成功
ssh -T git@gitee.com //成功

ssh -T github //失败
ssh -T git@github.com //成功
```

12， 配置了config 文件，添加私钥到本地 `ssh-add xxx` 这步可以不用执行，也可以成功

