# Install Oh-My-Zsh


# Foreword

Commonly used shell interpreters are `sh` and `bash`, but there are, of course, others such as `ksh`, `csh`, and `zsh`.

The tool we are about to install, **Oh My Zsh**, is an extension based on `zsh`. Similarly, there is **Oh My Posh** for Windows, both of which are extension toolkits for the original command line. They provide many rich features and offer various command-line styles, making our terminal more visually appealing.

GitHub：[oh my zsh](https://github.com/ohmyzsh/ohmyzsh)

# Prepare

Before installing Oh My Zsh, we need to install zsh and set it as our default shell.

{{< admonition type="info" open=true >}}
PS: In some Linux distributions (such as Kali) and macOS, zsh is the default shell interpreter, so no additional setup is required in those cases.
{{< /admonition >}}

## Install zsh

View all available shell interpreters and the current shell interpreter.

```bash
cat /etc/shells
echo $SHELL
```

{{< tabs >}}
{{% tab title="ubuntu & debian" %}}
```bash
sudo apt-get install zsh -y

# switch shell to zsh
chsh -s /bin/zsh
```
{{% /tab %}}
{{% tab title="centos & redhat & fedora" %}}
```bash
sudo yum install zsh -y

# switch shell to zsh
chsh -s /bin/zsh
```
{{% /tab %}}
{{< /tabs >}}

## install oh my zsh

There are several ways to install Oh My Zsh. Typically, you can use the installation script provided by the official source or manually clone the repository and configure it yourself. Regardless of the method, the git command is required. If you don't have git installed, you'll need to install it.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Edit `.zshrc`

```bash
cat > ~/.zshrc <<"EOF"
export ZSH="$HOME/.oh-my-zsh"
export TERM=xterm-256color

# My preferred command line style can be customized according to the official documentation.
ZSH_THEME="dallas"

plugins=(
  git # git plugin, provides many shortened git commands, available for viewing in the official documentation
  sudo # sudo plugin, double-tap the Esc key to automatically add sudo to the beginning of a command
  docker # docker plugin, provides excellent command suggestions for docker
  kubectl # kubectl plugin, provides excellent command suggestions for kubectl
  themes # theme plugin, allows you to change the current command line style by typing 'theme'
)

source $ZSH/oh-my-zsh.sh
setopt no_nomatch
EOF

source ~/.zshrc
```

After executing the above commands, you'll notice that the current command line is using Oh My Zsh. You can also configure the `~/.zshrc` file to set up more plugins, and you can download non-official plugins into the `~/.oh-my-zsh/custom/plugins` directory for use.

## Install Plugins

Two recommended plugins are:

- zsh-syntax-highlighting: Provides syntax highlighting for commands.
- zsh-autosuggestions: Suggests previously typed commands based on history.

Here's how to install these two plugins:

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions
```

Edit the ~/.zshrc file to activate these

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

At this point, the installation is complete. You can try out the new command line now.


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/install-oh-my-zsh/  

