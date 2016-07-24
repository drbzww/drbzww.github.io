---
title: 一个电脑对应两个Github帐户
date: 2016-07-10 19:32:13
tags:
---
#  问题引入
最近和一个老同学协作编写代码，所以就到Github上又注册一个Github账号（下面统一称作TechGroup），所以我写的代码就想同时向我自己的Github帐户(下面统一称作MyGithub)和TechGroup提交，这是就会出现一个问题，因为一个ssh key只能用在一个帐户中（当你将一个ssh key添加到两个帐户时，会提示ssh key已经被使用），接下来还会有一个问题如何给一个本地git资源库配置两个远程资源库。
#  解决第一个问题的步骤如下：
1. 首先为TechGroup帐户创建一个ssh key：
```
$ ssh-keygen -t rsa -C cytmxk@foxmail.com
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/lala/.ssh/id_rsa):~/.ssh/tech_rsa(此处填写rsa文件的路径)
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in yes.
Your public key has been saved in tech_rsa.pub.
The key fingerprint is:
fb:c4:b0:e0:47:fd:be:e0:fb:ea:73:ef:a8:29:d5:22 cytmxk@foxmail.com
The key's randomart image is:
+--[ RSA 2048]----+
|                |
|                 |
|                 |
|         .       |
|      . S ..     |
|     . oE=o..    |
|      . +o+..    |
|       ..+.+..   |
|         oOB=+o  |
+-----------------+
```

2. 通过如下命令列举 ssh-agent中包含的ssh-key：
```
$ ssh-add -l
2048 SHA256:4IGL3+7tT+lDdxONJlKxwQsjZ7JlB2U1936UMWU+1T8  (RSA)
```
可以看到我原来为MyGithub生成的ssh key。
执行如下命令将私钥tech_rsa添加的 ssh-agent：
```
$ ssh-add ~/.ssh/tech_rsa
$ ssh-add -l
2048 SHA256:4IGL3+7tT+lDdxONJlKxwQsjZ7JlB2U1936UMWU+1T8  (RSA)
2048 SHA256:uDP8CISrd8Os1vl+wnuPj4ldJaiq2XQnRk9FGKHhDfU /Users/chenyang/.ssh/tech_rsa (RSA)
192:~ chenyang$ 
```
可以看到ssh-add -l命令输出中多了一行，这一行就是tech_rsa私钥。

3. 配置.ssh/config
```
#多个ssh的配置 begin
Host github.com
    HostName github.com
    IdentityFile ~/.ssh/id_rsa
Host tech.github.com
    HostName github.com
    IdentityFile ~/.ssh/tech_rsa
#多个ssh的配置 end
```
4. 将创建生成tech_rsa.pub中的内容添加到TechGroup帐户中，添加的流程这里就不赘叙了，接下来测试一下是否添加成功(如果输出如下结果，表明添加成功)：
```
$ ssh -T git@tech.github.com
Hi techgroup01! You've successfully authenticated, but GitHub does not provide shell access.
```
注意上面的git@tech.github.com中的tech.github.com与config中的tech _rsa的Host保持一致。

# 给一个本地git资源库配置两个远程资源库，执行如下命令即可：
```
$ git remote add my_origin git@github.com:cytmxk/customview.git
$ git remote add tech_origin git@tech.github.com:techgroup01/customview.git
```
上面命令中要注意第二句中的远程资源库的Host要和config中相同，可以通过如下命令列举出当前库的所用远程资源库：
```
$ git remote -v
my_origin	git@github.com:cytmxk/customview.git (fetch)
my_origin	git@github.com:cytmxk/customview.git (push)
tech_origin	git@tech.github.com:techgroup01/customview.git (fetch)
tech_origin	git@tech.github.com:techgroup01/customview.git (push)
```