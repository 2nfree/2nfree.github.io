# 安装oh My Zsh


# 前言

普遍我们使用的shell解释器都是sh、bash这两种，但是除此之外当然还有其他的例如ksh、csh和zsh等

现在要安装的oh my zsh是基于zsh的一个扩展，与之相同的还有windows上的oh my posh，他们都是基于原始命令行上的一个扩展工具集，提供了很多丰富的功能，而且提供了很多的命令行样式，可以使我们的命令行更加好看

官方仓库地址：[oh my zsh](https://github.com/ohmyzsh/ohmyzsh)

# 准备工作

在安装oh my zsh之前，我们需要安装zsh，并且将zsh设置为我们的默认命令行

PS：在部分linux发行版（如kali）中包括新版的MAC OS操作系统，都是默认使用zsh作为shell解释器，像这种就不需要进行设置了

## 安装zsh

```bash
#查看所有的shell解释器
cat /etc/shells

#查看当前使用shell解释器
echo $SHELL
```

{{< tabs >}}
{{% tab title="ubuntu & debian" %}}
```bash
sudo apt-get install zsh -y

#切换shell到zsh
chsh -s /bin/zsh
```
{{% /tab %}}
{{% tab title="centos & redhat & fedora" %}}
```bash
sudo yum install zsh -y

#切换shell到zsh
chsh -s /bin/zsh
```
{{% /tab %}}
{{< /tabs >}}

## 安装oh my zsh

安装oh my zsh有很多方式，一般可以直接使用官方提供的安装脚本，或者自己clone仓库自行配置，但无论那种方式我们都需要git命令，如果没有这个命令则需要安装

{{< tabs >}}
{{% tab title="官方安装方式" %}}
```bash
#官方安装方式需要较好的网络环境
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
{{% /tab %}}
{{% tab title="镜像源安装" %}}
```bash
#从镜像源下载安装脚本
wget https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh
```

下载完安装脚本后，我们需要将安装脚本中的仓库地址改为镜像地址，主要修改下面两项

```shell
REPO=${REPO:-ohmyzsh/ohmyzsh}
REMOTE=${REMOTE:-https://github.com/${REPO}.git}

#改为
REPO=${REPO:-mirrors/oh-my-zsh}
REMOTE=${REMOTE:-https://gitee.com/${REPO}.git}
```

如图所示
{{< image src="repo.png" >}}

修改完后执行脚本，并切换oh my zsh的git远程仓库地址为镜像源，便于后续的更新

```bash
sh install.sh
cd ~/.oh-my-zsh
git remote set-url origin https://gitee.com/mirrors/oh-my-zsh.git
git pull
```
{{% /tab %}}
{{< /tabs >}}

## 编辑.zshrc

```bash
cat > ~/.zshrc <<"EOF"
export ZSH="$HOME/.oh-my-zsh"
export TERM=xterm-256color

ZSH_THEME="dallas" #个人比较喜欢的命令行样式，可以根据官方文档自行修改

plugins=(
  git #git插件，提供了很多git的缩减命令，可以在官方查看
  sudo #sudo插件，连按两下esc键自动在命令前加上sudo
  docker #docker插件，为docker提供了很好的命令提示
  kubectl #kubectl插件，为kubectl提供了很好的命令提示
  themes #皮肤插件，命令行输入theme可以更换当前的命令行样式
)

source $ZSH/oh-my-zsh.sh
setopt no_nomatch
EOF

#使用配置
source ~/.zshrc
```

在执行了上面的命令后，可以发现当前命令行使用了oh my zsh，我们还可以配置 ~/.zshrc 文件来设置更多的插件，也可以将除了官方以外的插件下载到 ~/.oh-my-zsh/custom/plugins 下来使用

## 安装插件

比较推荐的插件有两个：

- zsh-syntax-highlighting：命令高亮
- zsh-autosuggestions：历史命令提示

下面来安装这两个插件

{{< tabs >}}
{{% tab title="官方仓库安装" %}}
```bash
cd ~/.oh-my-zsh/custom/plugins
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
git clone https://github.com/zsh-users/zsh-autosuggestions
```
{{% /tab %}}
{{% tab title="镜像源安装" %}}
```bash
cd ~/.oh-my-zsh/custom/plugins
git clone https://gitee.com/lightnear/zsh-syntax-highlighting.git
git clone https://gitee.com/mattuy/zsh-autosuggestions.git
```
{{% /tab %}}
{{< /tabs >}}

之后我们再修改.zshrc文件来使用这两个插件，在plugins中添加刚才下载的插件即可

```bash
plugins=(
  git
  sudo
  docker
  kubectl
  themes
  zsh-autosuggestions
  zsh-syntax-highlighting
)
```

到此为止就已经安装完了，可以试试新的命令行了


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/install-oh-my-zsh/  

