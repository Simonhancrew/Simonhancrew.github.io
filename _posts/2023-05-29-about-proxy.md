---
title: Proxy Settings For SSH Git
date: 2023-05-29 14:10:00 +0800
categories: [Blogging, config]
tags: [writing]
---

### 1. Chrome Set Proxy

chrome配合SwitchOmega这个插件就可以做到通配符自定义代理了

普通情景模式选一个代理的地址填进去

自动切换情景模式里规则列表选择AutoProxy，然后更新地址为`https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt`，如果这个地址不行的话就用`https://pagure.io/gfwlist/raw/master/f/gfwlist.txt`.

这两个都被墙了就开proxy的情景模式拉一下，平时用自动情景模式就好了

### 2. Git Set Proxy

git本身支持的代理设置方式

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy https://127.0.0.1:7890
```

或者在bashrc里写一个函数，方便切换, 但是这个只能使用http协议，不能使用socks5协议。如果是在WSL里的话，需要多加几句

```bash
# wsl里面需要用HOST_IP替换localhost
HOST_IP=$(ip route show | grep -i default | awk '{ print $3}')
WSL_IP=$(hostname -I | awk '{print $1}')
PROXY_PORT=7890

PROXY_HTTP="http://${HOST_IP}:${PROXY_PORT}"
PROXY_SOCKS5="socks5://${HOST_IP}:${PROXY_PORT}"

# Host github.com
# User git
# ProxyCommand nc -X connect -x 0.0.0.0.0 %h %p

proxy(){
  # wsl里记得替换这里
  export http_proxy=http://127.0.0.1:7890
  export https_proxy=https://127.0.0.1:7890
  export all_proxy=socks5://127.0.0.1:7890

  # git config --global http.proxy ${PROXY_HTTP}
  # git config --global https.proxy ${PROXY_HTTP}

  # 这里要求设置~/.ssh/config里 # ProxyCommand nc -X connect -x 0.0.0.0:0 %h %p
  sed -i "s/# ProxyCommand/ProxyCommand/" ~/.ssh/config
  sed -i -E "s/ProxyCommand nc -X connect -x [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+ %h %p/ProxyCommand nc -X connect -x ${HOST_IP}:${PROXY_PORT} %h %p/" ~/.ssh/config

  echo "proxy on"
}

noproxy() {
  unset http_proxy
  unset https_proxy
  unset all_proxy
  # git config --global --unset http.proxy ${PROXY_HTTP}
  # git config --global --unset https.proxy ${PROXY_HTTP}
 
  sed -i -E "s/ProxyCommand nc -X connect -x [0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+ %h %p/# ProxyCommand nc -X connect -x 0.0.0.0:0 %h %p/" ~/.ssh/config
}
```

要支持ssh代理的话，需要在~/.ssh/config里面添加如下配置, 下面这个是在unix系的系统上使用

```bash
Host github.com
  User git
  ProxyCommand nc -X 5 -x 127.0.0.1:7890 %h %p
```

1. -X 5 表示使用socks5协议, 如果`connect`代表使用https
2. -x 之后的参数是代理服务器的地址和端口
3. %h %p 分别表示远程主机和端口，不要更改

### 2. Win下使用代理可能出现的一种情况的work around

如果在win下使用的话只需要把proxycommand稍做更改即可

```bash
ProxyCommand connect -S 127.0.0.1:7890 -a none %h %p
```

其中connect后的`-S`代表的是socks5，也可以选择`-H`, 代表https, `-a`代表的是认证方式，这里是none

另外我发现在windows下存在使用代理会有如下的fail case

```bash
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
fatal: Could not read from remote repository.
```

还没来得及研究原因，但我把`.ssh/config`里的和ssh.github.com的设置写成如下的就能过了:

```bash
Host ssh.github.com
  User git
  Port 443
  HostName ssh.github.com
  IdentityFile "~/.ssh/id_rsa"
  TCPKeepAlive yes
  ProxyCommand connect -S 127.0.0.1:7890 -a none %h %p
```

### REF 

1. [Accessing network applications with WSL](https://learn.microsoft.com/en-us/windows/wsl/networking#accessing-windows-networking-apps-from-linux-host-ip)
