##在 Mac OS 10.9 上安装 Fish Shell

[Fish](http://fishshell.com) 是一个不那么流行但是非常好用的交互式 Shell ，在 Mac OS 上可以使用 brew 命令进行安装，或者下载 [pkg](http://fishshell.com/files/2.1.0/fish.pkg) 文件进行安装。

```
brew install fish
```

然后修改 /etc/shells 文件，把 fish 加入到文件的最后。

```
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
/usr/local/bin/fish
```

修改 终端工具 的 偏好配置，在 Shell 的打开方式中选择命令，输入框中写入 /usr/local/bin/fish 。重启 终端工具 后可以 fish 就替代了默认的 bash 。

##安装 Docker 的命令补全

```
mkdir ~/.config/fish/completions
wget https://raw.github.com/barnybug/docker-fish-completion/master/docker.fish -O ~/.config/fish/completions/docker.fish
```

输入 docker 命令后按 tab 键就可以看到命令补全的输出，输入命令的前几个字母的时候也可以按 tab 查看补全的信息。

![Docker completion in fish shell](http://docker.u.qiniudn.com/docker-fish-shell-completion.png)




