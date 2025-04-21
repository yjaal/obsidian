
```sh

export ZSH=$HOME/ohmyzsh
ZSH_THEME="robbyrussell"



# jdk
JAVA_HOME=/Users/YJ/soft/jdk8/Contents/Home
# JAVA_HOME=/Users/YJ/soft/jdk11/Contents/Home
PATH=$JAVA_HOME/bin:$PATH:.
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.
export JAVA_HOME
export PATH
export CLASSPATH

# maven
export M2_HOME=/Users/YJ/soft/maven3.6.3
export PATH=$PATH:$M2_HOME/bin

# gradle
GRADLE_HOME=/Users/YJ/soft/gradle-6.7
export GRADLE_HOME
export PATH=$PATH:$GRADLE_HOME/bin

# 抽风了，需要加一下
PATH=/bin:/usr/bin:/usr/local/bin:${PATH}
export PATH

# 设置vscode启动的命令别名
alias code="/Applications/Visual\ Studio\ Code.app/Contents/Resources/app/bin/code"

# golang
# 参考：https://studygolang.com/articles/11081
export GOPATH=/usr/local/Cellar/go/1.24.2
export PATH=$PATH:$GOPATH/bin
export GOBIN=/usr/local/Cellar/go/1.24.2/bin
export PATH=$PATH:$GOBIN
export GOROOT=/usr/local/Cellar/go/1.24.2/libexec
export PATH=$PATH:$GOROOT/bin
export GO111MODULE=on

alias vscode="/Applications/Visual\ Studio\ Code.app/Contents/Resources/app/bin/code"
alias python=/usr/bin/python3

# python selenium chrome 驱动
# export PY_SELENIUM_DRIVER = /Users/YJ/study/python-work/driver
# export PATH=$PATH:$PY_SELENIUM_DRIVER


# zookeeper
export ZK_HOME=/Users/YJ/soft/zk-3.7.0
export PATH=$PATH:$ZK_HOME/bin
```


美化 vim

相关代码放在

```
/Users/YJ/.vim/bundle
```

编辑

```
～/.vimrc
```

```
set nocompatible
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
Plugin 'VundleVim/Vundle.vim'

" code complete
Plugin 'davidhalter/jedi-vim'
Plugin 'ervandew/supertab'

" syntastic check
Plugin 'nvie/vim-flake8'
Plugin 'scrooloose/syntastic'

" colorscheme
Plugin 'altercation/vim-colors-solarized'
Plugin 'luochen1990/rainbow' , {'for': 'python'}
Plugin 'morhetz/gruvbox'

" code format
Plugin 'mindriot101/vim-yapf'

" file search
Plugin 'ctrlpvim/ctrlp.vim'

call vundle#end()
filetype plugin indent on

colorscheme gruvbox

" for code complete
let g:jedi#auto_initialization = 1
let g:jedi#completions_enabled = 0
let g:jedi#show_call_signatures = 1


" for <leader>
let mapleader = ","
let g:mapleader = ","

" goto definition
let g:jedi#goto_definitions_command = ""
let g:jedi#goto_assignments_command = "<leader>g"
let g:jedi#goto_command = "<leader>d"

" file search
let g:ctrlp_map = '<c-p>'
let g:ctrlp_cmd = 'CtrlP'
" serach file in MRU
nmap <Leader>f :CtrlPMRUFiles<CR>
" search file in BUffer
nmap <Leader>b :CtrlPBuffer<CR>


set number
set cursorline
set fileencoding=utf-8
set fencs=ucs-bom,utf-8,cp936,gb18030,big5,euc-jp,euc-kr,latin1
set history=500
let python_highlight_all=1
set background=dark
set t_Co=256
set laststatus=2
set viminfo+=!
set showmatch
set matchtime=5
set ignorecase
set hlsearch
set autoindent
set cindent
set tabstop=4
set expandtab
set softtabstop=4
set shiftwidth=4
set autochdir
set autoread

highlight OverLength ctermbg=red ctermfg=white guibg=#592929
autocmd! FileType python match OverLength /\%89v.\+/
```

然后打开 vim，在 normal 模式下输入

