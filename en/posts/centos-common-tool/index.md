# Centos Common Tools


# EPEL (Extra Packages for Enterprise Linux)

EPEL (Extra Packages for Enterprise Linux) is a project based on Fedora, which provides extra packages for "Red Hat" operating systems, applicable to RHEL, CentOS and Scientific Linux.

When we use yum to install under Centos, we often can't find rpm, and the official rpm repository doesn't provide enough rpm packages, so it's a pain to compile them by yourself, but EPEL can solve these two problems, and EPEL is called Extra Packages for Enterprise Linux. EPEL is a project created by the Fedora community to provide high-quality packages for RHEL and its derivatives such as CentOS and Scientific Linux. When you install EPEL, you are adding a third-party source.

```bash
yum -y install epel-release
 
yum repolist
```

# Some Tools

- htop: is an interactive process browser for Linux that can be used to replace the top command under Linux.
- lsof: used to view the files opened by your process, the process that opened the file, the ports (TCP, UDP) opened by the process, and so on.
- ncdu (NCurses Disk Usage): a curses-based version of the famous "du" that provides a quick way to see which directories are using your disk space.
- Psmisc: package contains three programs that help manage the /proc directory. fuser displays the PIDs of the processes that are using the specified file or filesystem. killall kills a process of a certain name, which signals all the processes that are running the specified command. pstree tree displays the currently running processes.
- lrzsz: a program that replaces ftp for uploading and downloading in linux. rz uploads and sz downloads.
- iotop: a top-like tool that displays real-time disk activity.
- traceroute: route tracing
- vim: an editor, an upgraded version of vi
- wget: a command line tool for downloading files from the Web on Linux systems
- telnet: remote connection command
- tree: lists the tree structure of files
- xz, zip, unzip: compression and decompression tools

```bash
yum install htop lsof ncdu psmisc lrzsz iotop traceroute vim wget telnet tree xz zip unzip -y
```

# Development Tools

These tools include core development tools such as automake, gcc, perl, python, and debuggers which is required to compile software and build new rpms:

```bash
yum groups mark install "Development Tools"
yum groups mark convert "Development Tools"
yum groupinstall "Development Tools" -y
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/centos-common-tool/  

