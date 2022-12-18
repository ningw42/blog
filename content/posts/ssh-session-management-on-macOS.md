---
title: "macOS 中的 SSH Session 管理"
date: 2015-09-27T20:21:36+08:00
tags: [macOS, SSH]
---

之前一直使用 Windows，所以一直都能用到 XShell 或者 Putty 这样的软件，在登录 SSH 的同时还能管理 SSH Session。切换到 macOS 下却有点蛋疼，虽然 SSH 命令没有 Session 管理的功能，但是略微配置一下就能变的比 Windows 好用不少。

配置完成后可以做到：

1. 存储 Session
2. 使用 SSH Key 免密码登录
3. Session 自动补全

# 存储 Session

编辑 ~/.ssh/config 文件

```
Host dev
    HostName example.com
    User username
```

详细的选项可以在 /etc/ssh/ssh_config 中查看。现在在 ssh 命令后面直接跟上别名就好了。

# 使用 SSH Key 免密码登录

SSH Key 分为公钥和私钥，公钥放在被 SSH 的机器上，私钥自己拿着。

```bash
ssh-keygen -t rsa
```

使用上面的命令生成一对密钥，可以自己取名字，建议在 ~/.ssh/ 目录下执行。生成的两个文件中，后缀为pub的就是公钥，有多百种方法可以把公钥放倒服务器上，scp, sftp等等，但是要注意authorized_keys存在与否的区别。也有一键式的命令 ssh-copy-id 来复制 SSH Key

```bash
# scp
scp public_key username@example.com:~/.ssh/authorized_keys

# ssh-copy-id
ssh-copy-id username@example.com
```

然后继续编辑刚才的 ~/.ssh/config 文件，在想要使用 SSH Key 登录的 Host 下加上

```sh
	PreferredAuthentications publickey
	IdentityFile ~/.ssh/private_key_name
```

现在登录就不需要密码了，可以禁用密码登录来提高安全性，前提是保证 SSH Key 的安全。编辑服务器上的 /etc/ssh/sshd_config 并重启 ssh 服务

```sh
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```

# Session 自动补全

貌似 zsh 自带这个功能，如果使用 bash 可以使用[这里](http://www.commandlinefu.com/commands/view/2759/ssh-autocomplete)的方法来补全别名。


