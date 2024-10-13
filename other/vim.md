# neovim 配置安装

## 0x00 源

ubuntu22直接可以这样安装：
```shell
sudo apt-get install neovim
```

## 0x01 配置文件

nvim的配置目录跟vim有一些不一样，建议是卸载一个，我这里保留了nvim，其配置地址在`$(pwd)/.config/nvim/init.vim`：

```vim
set cursorline
set number
call plug#begin('~/.local/share/nvim/plugged')
Plug 'neoclide/coc.nvim', {'branch': 'release'}
call plug#end()
```
其中`~/.local/share/nvim/plugged`则是这个vimplug插件下载后插件所在的地址。

上面的配置文件安装了了一个coc.nvim插件，还需要在其中安装一下cocjson插件才能使用：

```vim
:CocInstall coc-json coc-tsserver
:CocConfig
```

就可以进行配置了，例如：

```vim
{
  "languageserver": {
    "clangd": {
      "command": "clangd",
      "filetypes": ["c", "cpp"],
      "rootPatterns": [".clangd", "compile_commands.json"],
      "initializationOptions": {
        "cache": {
          "directory": "/tmp/coc-clangd"
        },
        "clangd": {
          "fallbackFlags": ["-std=c++11"],
          "resourceDirectory": "/path/to/clangd/resources"
        }
      }
    }
  }
}
```

其中rootPatterns会在当前vim的打开路径下寻找对应的内容。

#### E117错误

MacOS中安装完Plug后，应该通过`:echo &rtp`得到对应的include路径，然后再将`plug.vim`放到对应的目录中。

比如我这里通过`:echo &rtp`获得了一个路径`.config/nvim`，那么我就把`~/.config/nvim/plug.vim`放到

`~/.config/nvim/autoload/plug.vim`中，否则就会出现E117错误。



### 一些配置

```c++
set tabstop=2
set softtabstop=2
set shiftwidth=2
set expandtab
set number
set autoindent
// color x-code主题
colorscheme xcodedark
```



### 插件

```
Plug 'scrooloose/nerdtree' // 文件管理
Plug 'luochen1990/rainbow'	// 彩色括号
Plug 'arzg/vim-colors-xcode' // xcode主题
```









## vim8.0

插件放的地址：

```shell
~/.vim/pack/${name}/start
# like
~/.vim/pack/swagger/start
```

安装方式有在vim中直接安装，比如：

```shell
:helptags ~/.vim/pack/${name}/start/${pack_name}/doc
```

如果有这种错误：

 ```shell
 Cannot open /usr/share/vim/site/doc/tags for writin
 ```

可以尝试在安装开头加上：

```shell
silent!
```

__你也可以创建出一个opt的目录，和start处于相同目录下，start用于存放自动加载的插件，而opt用来存放手动载入的，当然手动载入的脚本也可以写到`.vimrc`中变为自动载入__ 。

### some plug

#### nerdtree

```shell
https://github.com/preservim/nerdtree
```

#### airline

```shell
https://github.com/vim-airline/vim-airline
```

#### onedark主题

```shell
https://github.com/joshdick/onedark.vim
```

安装方式：

将目录克隆到opt下，然后使用.vimrc(~/.vimrc)插件去加载：

```shell
packadd! onedark.vim
colorscheme onedark
```



## 快捷键

目前我是直接设定到`.vimrc`中，一些基础的快捷键：

* n表示normal模式生效
* i表示insert模式下生效

对应的一些按键：

* \<C> 表示ctrl， \<C-j>就表示ctrl + j这种，以此类推。
* \<CR>表示enter
* <>



### 分屏

水平分屏：

```shell
:sp 
```

垂直分屏：

```shell
:vsp
```

分屏的切换操作：

```shell
ctrl + w + h/j/k/l
```



## LSP



### 代码补全

使用到的是coc.vim插件：

```shell
https://github.com/neoclide/coc.nvim
```

可以直接放到start目录下，使用

```shell
:CocConfig
```

可以打开配置文件就证明成功了。



安装nodejs

需要更新的版本：

```shell
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
```



```shell
sudo apt-get install -y nodejs npm
```

```shell
sudo npm install -g yarn
```





#### error

```shell
/usr/bin/ld: cannot find -ltinfo
```

在centos上需要：

```shell
ncurses-devel
```

在ubuntu上需要：

```shell
sudo apt-get install libncurses5-dev
```













