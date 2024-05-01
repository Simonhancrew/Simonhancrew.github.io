---
title: blog theme
date: 2024-05-01 17:42:00 +0800
categories: [Blogging, blog-theme]
tags: [writing]
---

换了一个blog theme

## EnvPrepare

白嫖github page，然后找一个好看的主题就行了，主题我用的是[jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy/)

然后就是改Front Matter，之前那个的和现在的不太一样，我写的blog有点多，所以写了个脚本替换，不过就是改了下categories + date，然后title把"都去了

环境在本地搭了一下，直接ubuntu的docker镜像，然后安装ruby，jekyll，然后就可以本地预览了

```bash
sudo apt-get install ruby-full build-essential zlib1g-dev

echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

gem install jekyll bundler
```

然后配合他的[getting started](https://chirpy.cotes.page/posts/getting-started/)随便坐下check。

### 问题

就遇到了一个问题，GitHub workflow有个test site，htmlproofer检查过不了，[issue](https://github.com/cotes2020/jekyll-theme-chirpy/issues/1403)里提到了，我直接禁用了看起来也能work

```bash
# - name: Test site
#   run: |
#     bundle exec htmlproofer _site \
#       \-\-disable-external=true \
#       \-\-ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

还有一个就是wsl里起的container jekyll server没法在win下看localhost:4000，我映射了端口也不行，母鸡为啥。
