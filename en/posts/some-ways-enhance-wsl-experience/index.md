# Some Ways to Enhance Your WSL Experience


This article provides tips to help frequent WSL users improve their experience and efficiency. This article will not cover the installation of WSL, you can refer to the official documentation if you need it.
BTW I prefer to use the [WSL2](https://learn.microsoft.com/en-us/windows/wsl/) + [ArchWSL](https://github.com/yuk7/ArchWSL)

# Open wsl folders with Windows Explorer

Sometimes, we need to open files in WSL using Windows File Explorer. Normally, we can manually access Windows File Explorer and type ```\\wsl$``` in the address bar to navigate to WSL. However, if we want to quickly open a WSL directory from the command line (similar to how we use code + directory with VS Code's WSL Remote), we can achieve this by writing an executable script.
In Windows, we can use the explorer.exe command to open File Explorer, leveraging it to quickly open the desired WSL directory.
Scripts：

```bash
# wslpath -w "$1" Used to convert a user-entered directory into a shared address for WSL on the host machine
explorer.exe "$(wslpath -w "$1")" || true
```

Add it as a script and put it under /usr/local/bin, e.g. /usr/local/bin/open, and give it execute permission to use it quickly.
This allows you to quickly open a folder with a command line

```bash
open /data
```

# Network Proxy

There are two ways to set up a network proxy in WSL

## Type 1 Configuration using proxy ENV (http_proxy/https_proxy/all_proxy)

First, enable the "Allow LAN connections" option in your proxy software.
The IP address of the Windows WSL network interface can be obtained by running the command ```grep "nameserver" /etc/resolv.conf | head -n 1 | cut -f 2 -d " "```, and it can be used as the LAN connection address for WSL.
You can configure the environment variables using a script like the following ```proxy.sh on|off``` .

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

Alternatively, you can use the alias command to configure this in your bashrc or zshrc file.

```bash
WIN_IP=$(grep "nameserver" /etc/resolv.conf | head -n 1 | cut -f 2 -d " ")
PROXY_PORT=7890
alias proxyon="export https_proxy=http://$WIN_IP:$PROXY_PORT http_proxy=http://$WIN_IP:$PROXY_PORT all_proxy=socks5://$WIN_IP:$PROXY_PORT"
alias proxyoff="unset https_proxy http_proxy all_proxy"
```

## Type 2 Using the Tun Mode of the Proxy Software

If your proxy software supports Tun mode, we can enable it directly.
This ensures that all traffic from WSL is routed through the proxy software's virtual network adapter.


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/some-ways-enhance-wsl-experience/  

