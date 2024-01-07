---
title: Install Vim8.2 on Ubuntu 18.04 with Gtk3
slug: 2020-04-17-install-vim8.2-on-ubuntu-18.04-with-gtk3
date:  2020-04-17T14:00:00+08:00
draft: false
categories:
  - Vim
---


# 卸载原来的vim

```sh
sudo apt-get remove --purge vim vim-runtime gvim vim-tiny vim-common
```

# 安装build vim所需工具和库

<!--more-->

```sh
sudo apt install libncurses5-dev libgnome2-dev libgnomeui-dev \
libgtk2.0-dev libgtk-3-dev libatk1.0-dev libbonoboui2-dev \
libcairo2-dev libx11-dev libxpm-dev libxt-dev python-dev \
python3-dev ruby-dev lua5.1 liblua5.1-dev libperl-dev git


```

# 下载源代码

```sh
git clone https://github.com/vim/vim.git
cd vim
```

# 配置编译

需要将 `--with-python3-config-dir` 改成自己机器上的路径。
使用 `--enable-fail-if-missing` 是为了在缺少某些库报错，而不是忽略。
其他参数含义见 `./configure --help`。

```sh
./configure --with-features=huge \
            --enable-multibyte \
            --enable-rubyinterp=yes \
            --enable-python3interp=yes \
            --with-python3-config-dir=/usr/lib/python3.6/config-3.6m-x86_64-linux-gnu \
            --enable-perlinterp=yes \
            --enable-luainterp=yes \
            --enable-gui=gtk3 \
            --enable-fail-if-missing \
            --prefix=/usr/local
```

# 编译

```sh
make VIMRUNTIMEDIR=/usr/local/share/vim/vim82
sudo make install
make clean
```

----
参考 http://www.gmloc.me/55.html
