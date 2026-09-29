# 提高 WSL 使用体验的小方法


本文旨在给经常使用 WSL 的兄弟们一些能够提高使用体验、提升效率的一些小技巧。在本文不会涉及到 WSL 的安装部分，有此需求可参考官方文档
参考：
[WSL2](https://learn.microsoft.com/en-us/windows/wsl/) + [ArchWSL](https://github.com/yuk7/ArchWSL)

# 使用 Windows 资源管理器快速打开 wsl 中的文件夹

有些时候我们有需要通过 Windows 资源管理器打开 wsl 中文件的需求，正常情况下我们可以手动访问 Windows 的资源管理器并在地址栏输入 ```\\wsl$``` 访问，但是如果我们想通过一个 command line 的形式快速打开（类似于使用 vscode wsl remote 中的 code + 目录的方式），我们就可以通过编写一个可执行脚本的方式实现
在 Windows 中，我们可以通过命令行 ```explorer.exe``` 打开资源管理器，所以我们可以使用它快速打开我们想要的 WSL 的目录
具体脚本如下：

```bash
# wslpath -w "$1" 用于将用户输入的目录转换成 WSL 在宿主机的共享地址
explorer.exe "$(wslpath -w "$1")" || true
```

将其添加为脚本并放在 /usr/local/bin 下，如 /usr/local/bin/open，给予执行权限后即可快速使用
如：

```bash
open /data
```

这样就可以通过 command line 的形式快速打开文件夹了

# 快速配置代理

WSL 中设置代理有两种方式

## 使用代理环境变量（http_proxy/https_proxy/all_proxy）配置

首先开启代理软件中的“允许局域网连接”
我们通过命令 ```grep "nameserver" /etc/resolv.conf | head -n 1 | cut -f 2 -d " "``` 可以获取 Windows 的 WSL 网卡地址，这个就可以作为 WSL 的局域网连接的地址
使用如下脚本实现环境变量的配置，如 proxy.sh on|off

```bash
#!/bin/bash
WIN_IP=$(grep "nameserver" /etc/resolv.conf | head -n 1 | cut -f 2 -d " ")
PROXY_PORT=7890
if [ "$1" == "on" ]; then
    export https_proxy="http://$WIN_IP:$PROXY_PORT"
    export http_proxy="http://$WIN_IP:$PROXY_PORT"
    export all_proxy="socks5://$WIN_IP:$PROXY_PORT"
    echo "Proxy is set to http://$WIN_IP:$PROXY_PORT"
elif [ "$1" == "off" ]; then
    unset https_proxy http_proxy all_proxy
    echo "Proxy is unset"
else
    echo "Usage: $0 {on|off}"
fi
```

或者可以使用 alias 命令在 bashrc or zshrc 中配置

```bash
WIN_IP=$(grep "nameserver" /etc/resolv.conf | head -n 1 | cut -f 2 -d " ")
PROXY_PORT=7890
alias proxyon="export https_proxy=http://$WIN_IP:$PROXY_PORT http_proxy=http://$WIN_IP:$PROXY_PORT all_proxy=socks5://$WIN_IP:$PROXY_PORT"
alias proxyoff="unset https_proxy http_proxy all_proxy"
```

## 使用代理软件的 Tun 模式

如果我们的代理软件支持 Tun 模式，我们可以直接开启 Tun 模式，这样 WSL 的流量就会被转发到代理软件的虚拟网卡上了


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/some-ways-enhance-wsl-experience/  

