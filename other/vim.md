## NEOVIM踩坑



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

