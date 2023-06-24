---
title: zsh settings
author: simon
date: 2023-06-24
category: config
layout: post
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


