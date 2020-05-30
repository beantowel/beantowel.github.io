---
layout: post
title:  "如何找回ssh公钥？"
date:   2020-05-31 00:19:01 +0800
categories: linux
---
## 公钥怎么没了
为什么需要找回公钥呢？因为干了一件蠢事，因为ssh登陆服务器懒得输入密码，就准备把本机公钥`~/.ssh/id_rsa.pub`添加到服务器的授权密钥`~/.ssh/authorized_keys`里去，结果因为是临时随便搜的方法，拷贝的时候用的这个命令：`scp ~/.ssh/id_rsa.pub 登录用户名@服务器域名:～/.ssh/`，按下回车我就后悔了，这不是把服务器的公钥覆盖了么。赶紧测试一下是不是这样，比如[Github](http://github.com)一般都是设置ssh key登陆账号的，测试连接：

```bash
$ ssh -T git@github.com
git@github.com: Permission denied (publickey).
```

果然出事了。

## 找回ssh公钥
意外的简单：

```bash
$ ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
```

注意这样生成的公钥是没有注释的，一般在后面补上 空格+用户名@主机名 更好。

## 为什么能找回公钥
之前我还在担心无法找回，毕竟记得rsa是个对称的加密算法，能从私钥找回公钥岂不是反向也可？
赶紧补补课，比较重要的点是这几个：
* 什么是公钥？两个数(n, e)
* 私钥？两个数(n, d)
* n=p*q 是一个不容易分解的大质数
* d⋅e ≡ 1 (mod λ(n)) d和e互为模λ(n)的逆，λ(n) 是 Carmichael's totient function
* 没有分解出n的情况下λ(n)难以求解，反之已知p和q时则λ(n) = lcm(p − 1, q − 1)，求个最大公因数即可。

所以，光有公钥或私钥都是不能直接计算出对方的，而rsa加密初始计算完成后只需要保留公钥私钥。但是，本机上在文件`id_rsa`其实保存了私钥公钥和生成密钥时的其他所有数据，用

```bash
$ ~/.ssh$ openssl rsa -text -noout < id_rsa
```

就能方便地查看，其中就有modulus，publicExponent，privateExponent，prime1，prime2等等，对应于n, e, d, p, q。而且通常来说公钥都是固定的一个数字65537 (0x10001)，所以找回公钥非常方便。


