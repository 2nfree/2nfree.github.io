# Centos Common Tools


# EPEL (Extra Packages for Enterprise Linux)

EPEL (Extra Packages for Enterprise Linux)：是基于Fedora的一个项目，为“红帽系”的操作系统提供额外的软件包，适用于RHEL、CentOS和Scientific Linux.

我们在Centos下使用yum安装时往往找不到rpm的情况，官方的rpm repository提供的rpm包也不够丰富，很多时候需要自己编译很痛苦，而EPEL恰恰可以解决这两方面的问题。EPEL的全称叫 Extra Packages for Enterprise Linux 。EPEL是由 Fedora 社区打造，为 RHEL 及衍生发行版如 CentOS、Scientific Linux 等提供高质量软件包的项目。装上了 EPEL之后，就相当于添加了一个第三方源。

```bash
yum -y install epel-release
 
yum repolist
```

# 好用的工具

- htop：是一个Linux下的交互式的进程浏览器，可以用来替换Linux下的top命令。
- lsof：用于查看你进程打开的文件，打开文件的进程，进程打开的端口(TCP、UDP)等。
- ncdu（NCurses Disk Usage）：是著名的“du”的基于curses的版本，它提供了一种快速查看哪些目录正在使用您的磁盘空间的方法。
- Psmisc：软件包包含三个帮助管理/proc目录的程序。fuser 显示使用指定文件或者文件系统的进程的PID。killall 杀死某个名字的进程，它向运行指定命令的所有进程发出信号。pstree 树型显示当前运行的进程。
- lrzsz：在linux里可代替ftp上传和下载的程序，使用rz上传，sz下载
- iotop：是一个类似 top 的工具,用来显示实时的磁盘活动
- traceroute：路由跟踪
- vim：编辑器，vi升级版
- wget：是Linux系统用于从Web下载文件的命令行工具
- telnet：远程连接命令
- tree：列出文件树形图结构
- xz、zip、unzip：压缩及解压工具

```bash
yum install htop lsof ncdu psmisc lrzsz iotop traceroute vim wget telnet tree xz zip unzip -y
```

# Development Tools

这些工具包括核心开发工具，如 automake、gcc、perl、python 和调试器，它们是编译软件和构建新 rpms 所必需的

```bash
yum groups mark install "Development Tools"
yum groups mark convert "Development Tools"
yum groupinstall "Development Tools" -y
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/centos-common-tool/  

