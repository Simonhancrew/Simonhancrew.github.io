---
title: zsh settings
date: 2023-06-24 14:10:00 +0800
categories: [Blogging, config]
tags: [writing]
---

## terminal settings

最简单的办法当然是直接安装zsh + ohmyzsh。虽然这样在插件较多的时候会牺牲一点启动的速度.

安装zsh + ohmyzsh
```bash
sudo zypper install zsh
# sudo apt install zsh
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

然后安装插件，我这边在wsl里，就需要一个自动补全就够了

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

vim .zshrc
# plugins=( 
#     # other plugins...
#     zsh-autosuggestions
# )
```

## 遇到一些很傻逼的问题

如果遇到一些比较傻逼的问题的话，首先先进.oh-my-zsh里去，把里面修改过的文件restore了，然后source一把zshrc

还不行的话就需要开debug mode复现一下了，omz开debug比较简单，看这个[页面](https://github.com/ohmyzsh/ohmyzsh/wiki/Troubleshooting#other-problems)就可以了
