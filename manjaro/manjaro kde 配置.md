# 基本配置
## 先换源：
```shell
sudo pacman-mirrors -i -c China -m rank && sudo pacman -Syyu
```

如果有签名错误：
```shell
sudo pacman -S archlinuxcn-keyring
sudo pacman -S antergos-keyring
```

并将etc/pacman.conf中的 `SigLevel = Optional TrustAll`

## 安装配置clash
### clash配置 
[clash下载](https://github.com/Dreamacro/clash/releases)
解压gz: `gunzip`
wget -O config.yaml `your link`
http和https代理设置为127.0.0.1:7890

### 命令行代理
在~/.zshrc末尾添加
```shell
function proxy_on() {
    export http_proxy=http://127.0.0.1:7890
    export https_proxy=$http_proxy
    echo -e "终端代理已开启。"
}

function proxy_off(){
    unset http_proxy https_proxy
    echo -e "终端代理已关闭。"
}
```

`source ~/.zshrc`
在命令行需要代理是通过 `proxy_on` 开启命令行代理。

## 安装yay
```shell
cd /opt
sudo git clone https://aur.archlinux.org/yay.git
sudo chown -R 'group:user' ./yay
cd yay && makepkg -si
```

## 安装中文输入法
```shell
sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons fcitx5-material-color
```
将下面内容粘贴到`~/.pam_environment`（没有就新建）
```
GTK_IM_MODULE DEFAULT=fcitx
QT_IM_MODULE  DEFAULT=fcitx
XMODIFIERS    DEFAULT=@im=fcitx
```

### 系统登陆后默认启动fcitx5输入法
将下面的内容粘贴到 `~/.xprofile`（没有就新建）
```
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
fcitx5 &
```

### 配置主题
使用`fcitx5-material-color`这个主题,可以参照: [Fcitx5-Material-Color](https://github.com/hosxy/Fcitx5-Material-Color)

# 软件安装
## Clion安装
```shell
sudo pacman -S snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
sudo snap install clion --classic
```

# 建立回收站
```shell
mkdir ~/.trash
```
配置 `bashrc`或者`zshrc`
```shell
alias rm=trash
alias r=trash
alias rl='ls ~/.trash/'
alias ur=recoverfile

recoverfile()
{
    mv -i ~/.trash/$@ ./
}

trash()
{
    mv $@ ~/.trash
}
```

更新配置文件：
```shell
source ~/.zshrc
```

