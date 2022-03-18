---
title: 如何管理多个 git 账号
date: 2021-08-15
categories:
 - Git
tags:
 - Git
 - SSH
---

## 前言

一般来说，在公司做项目用的 git 账号是公司分配的，而我们维护自己的 github 使用的是自己的 git 账号，并且项目所存放的平台是不一样的，因此当我们在同一台电脑上进行操作时，如果配置不正确，就会产生冲突，甚至无法提交修改记录。

git 可以使用四种不同的协议来传输资料，最常用的就是 HTTPS 和 SSH，接下来会分别说明。

## HTTPS

使用 HTTPS 去 clone 一个仓库是没有限制的，这也是一般开源项目推荐的方式。虽然说拉取下来没有限制，但是在 push 时会让我们输入用户名和密码以验证权限，而且每次 push 都需要输一遍用户名和密码，像这样：![git-push-username](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/git-push-username.png)

这种操作很繁琐，原因是提交的时候没有自动将身份信息交给服务器去验证，需要我们手动去做，所以我们要做的就是做一些配置，能够使得这个验证步骤在后台自动去完成。

git 提供了一个命令，可以让我们把用户名和密码存储在本地，减少我们输入用户名和密码的次数。

> git config credential.helper 'store [<options>]'

options 中可以使用 --file 指定需要存储的位置，如果没有指定，则默认存储在 ~/.git-credentials。需要注意的是密码是未加密的方式存储，需要保管好。

在仓库根目录执行完上述命令后，我们继续 push 操作，依然会让我们输入用户名和密码，如果提交成功，那么此次输入的用户名和密码就会存在上面指定的位置，下次 push 时会去存储位置查询用户名和密码进行验证，也就不需要输入用户名和密码了。

如果使用的是 github 仓库，在 push 时有可能会遇到如下错误：
> Support for password authentication was removed on August 13, 2021. Please use a personal access token instead
> Please see https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/ for more information

意思就是从 2021年8月13号开始，上述密码验证的方式被移除了，取代的是使用 token 验证，具体通过上述链接进入官网查看。总的来说， token 验证方式更接近 SSH。

## SSH

相较于 SSH，HTTPS 的方式会有如下缺点：
- 需要输入用户名和密码
- 保存在本地的密码是未加密的
- 由于网络的原因，操作经常会报 443 time out 等问题

但使用 SSH 依然需要一个验证过程，用到的是 SSH key，它是一对密钥包括公钥和私钥，私钥保存在本地，公钥配置在远程仓库。因此在拉取代码之前就需要将配置好公钥，以便拉取的时候进行验证。

首先使用 ssh-keygen 创建密钥对
> ssh-keygen -t rsa -C "your_email@example.com"

- -t rsa 意思是使用 rsa 算法来创建
- -C "your_email@example.com" 就是将一个字符串添加为签名，通常为邮箱

一路回车之后，默认会在 "/c/Users/Administrator/.ssh/" 位置生成两个文件，id_rsa 和 id_rsa.pub，分别是私钥和公钥。

然后就需要将私钥添加到 ssh-agent

第一步启动
> eval "$(ssh-agent -s)"

第二步添加
> ssh-add ~/.ssh/id_rsa

添加完成之后，就可以将公钥配置在 github 或其他平台。可以参考[github 公钥配置](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)。

到这里大功告成，可以使用 SSH 协议拉取代码，推送修改了。

### 添加新账号

因为公司最近决定从 svn 切换到 git 去管理公司代码，在内网搭建的 gitlab 服务器，并且分配了邮箱。此时我需要管理两个 git 账号：

- 存在全局的，用于我自己 github
- 公司分配的，用于某一个项目

当我第一次给公司项目推送修改时，我看到我个人 git 用户名出现在 commit 里（这个项目应该是没有做权限），当时就觉得事情不对，按正常逻辑来讲，这个推送应该是要被驳回的，问题提示应该是这样：
> Permission to user1/example.git denied to user2

意思是我在做 push 操作时使用的是 user2，但是 user2 没有 user1/example.git 这个仓库的权限。只不过该没有设置权限，直接给通过了。那就算没有设置权限，也不能使用个人的账号，所以我就在公司项目根目录配置了分配的账号，那后面的推送就是使用公司分配的名称，需要注意的是这里不加上 --global，代表当前仓库。

> git config user.name "xxx"
> git config user.email "xxx@hc.com"

可以在仓库的 config 文件中查看

这个问题算是解决了，但是病根并没有解除，那就是多账号冲突的问题。解决办法也很简单，那就是按上面步骤用这个新账号创建新的 SSH key。
唯一不同的是在询问密钥存放位置的时候，需要设置成新的，例如可以这样：
> Enter file in which to save the key (/c/Users/Administrator/.ssh/id_rsa): /c/Users/Administrator/.ssh/xxx_hc_rsa

意思就是创建新的密钥对 xxx_hc_rsa 和 xxx_hc_rsa.pub，添加私钥和公钥依然跟上面的操作一样。

现在就存在多个私钥进行管理，需要在 "/c/Users/Administrator/.ssh/" 配置 config 文件，如果没有就创建一个，配置如下

```js
# user2
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile C:/Users/Administrator/.ssh/id_rsa
User user2@qq.com

# xxx_hc
Host gitlab.com
HostName gitlab.com
PreferredAuthentications publickey
IdentityFile C:/Users/Administrator/.ssh/xxx_hc_rsa
User xxx@hc.com
```

配置完成之后，每次操作，不同域名仓库会从这个文件中拿到对应的私钥进行加密验证。

### 两个 github 账号

但万万没想到，我还有一个 github 账号，这个账号是专门用来做测试的，从来没有在这电脑上使用。当我在这个电脑上进行测试时，就出现了上面提到的那个错误：
> Permission to user1/example.git denied to user2

按照上面说的，无非就是再加一个 git 账号，于是依然按照上面的步骤进行操作，最后我的 config 文件变成这样：

```js
# user2
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile C:/Users/Administrator/.ssh/id_rsa
User user2@qq.com

# xxx_hc
Host gitlab.com
HostName gitlab.com
PreferredAuthentications publickey
IdentityFile C:/Users/Administrator/.ssh/xxx_hc_rsa
User xxx@hc.com

# user1 // 不同点                                   
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile C:/Users/Administrator/.ssh/id_rsa_user1 // 不同点
User user1@qq.com // 不同点
```

配置完成之后，我再次操作，依然还是提示上述问题。创建密钥，添加密钥这些步骤都是固定的，肯定没什么问题，仔细检查这个 config 文件会发现，两个 github 账号使用的 Host 都是 github.com，前面有说过不同域的仓库会从中拿到对应的私钥，会不会存在匹配到了第一个，然后使用的第一个私钥呢？于是我将 user1 这个配置移动到文件顶部，发现果然通过了，看来确实是这个问题导致的。

但是 github.com 这个域名是固定的，怎么做一下区分呢？

这里的解决办法是添加子域名，我们可以将 github 里的，每个仓库它属于 github 哪个子域名下，也就是将第三个配置做以下修改：
> Host user1.github.com

config 做修改之后，还需要修改仓库里的配置，将 origin 修改成 config 中一样。

```js
[remote "origin"]
	url = git@github.com:user1/blog.git
```

改成：

```js
[remote "origin"]
	url = git@user1.github.com:user1/blog.git
```

这样处理之后，按照不同域获取不同的私钥就不会拿错了。

## 最后

1. 通常会使用 HTTPS 和 SSH 这两种协议关联远程仓库，HTTPS 使用用户名和密码去验证，而 SSH 通过密钥对验证。
2. HTTPS 的方式管理一个和多个账号方式是一样的，但是 SSH 则需要注意以下几点：
   - 私钥存放本地，公钥配置在代码托管平台
   - 需要将私钥添加到 ssh-agent 中
   - 不同的代码托管平台配置不同的 Host
   - 相同的 Host 可以添加子域名以作区分

参考文献：

https://qastack.cn/programming/3225862/multiple-github-accounts-ssh-config

https://blog.csdn.net/PeipeiQ/article/details/80702514
