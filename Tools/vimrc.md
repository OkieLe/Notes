Vim
```vim
set showcmd
set showmatch
set matchtime=5
set incsearch

set nu
set cursorline

set foldenable
set foldmethod=manual

set nocompatible
set history=100

set confirm

filetype on
filetype plugin on
filetype indent on

syntax on

set nobackup

set linespace=0
set wildmenu

set ruler
set rulerformat=%20(%2*%<%f%=\ %m%r\ %3l\ %c\ %p%%%)

set cmdheight=2
set backspace=2

set statusline=%F%m%r%h%w\[POS=%l,%v][%p%%]\%{strftime(\"%d/%m/%y\ -\ %H:%M\")}
set laststatus=2

set autoindent
set smartindent

set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab

if has("autocmd")
autocmd FileType xml,html,c,cs,java,perl,shell,bash,cpp,python,vim,php,ruby set number
autocmd FileType xml,html vmap <C-o> <ESC>'<i<!--<ESC>o<ESC>'>o-->
autocmd FileType java,c,cpp,cs vmap <C-o> <ESC>'<o
autocmd FileType html,text,php,vim,c,java,xml,bash,shell,perl,python setlocal textwidth=100
autocmd Filetype html,xml,xsl source $VIMRUNTIME/plugin/closetag.vim
autocmd BufReadPost *
\ if line("'\"") > 0 && line("'\"") <= line("$") |
\ exe " normal g`\"" |
\ endif
endif "has("autocmd")
```

MacVim
```
" Configuration file for vim
set modelines=0     " CVE-2007-2438

" Normally we use vim-extensions. If you want true vi-compatibility
" remove change the following statements
set nocompatible    " Use Vim defaults instead of 100% vi compatibility
set backspace=2     " more powerful backspacing

" Don't write backup file if vim is being called by "crontab -e"
au BufWrite /private/tmp/crontab.* set nowritebackup nobackup
" Don't write backup file if vim is being called by "chpass"
au BufWrite /private/etc/pw.* set nowritebackup nobackup

set showcmd
set showmatch
set matchtime=5
set incsearch

set nu
set cursorline

set foldenable
set foldmethod=manual

set nocompatible
set history=100

set confirm

filetype on
filetype plugin on
filetype indent on

syntax on

set nobackup

set linespace=0
set wildmenu

set ruler
set rulerformat=%20(%2*%<%f%=\ %m%r\ %3l\ %c\ %p%%%)

set cmdheight=2
set backspace=2

set statusline=%F%m%r%h%w\[POS=%l,%v][%p%%]\%{strftime(\"%d/%m/%y\ -\ %H:%M\")}
set laststatus=2

set autoindent
set smartindent

set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab

" PLUGIN

call plug#begin()
" a Git wrapper so awesome
Plug 'tpope/vim-fugitive'
" A tree explorer plugin for vim
Plug 'scrooloose/nerdtree'
" Git flag on nerdtree
Plug 'Xuyuanp/nerdtree-git-plugin'
" Plug 'kballard/vim-swift'
Plug 'bumaociyuan/vim-swift'
" Fuzzy file, buffer, mru, tag, etc finder
Plug 'kien/ctrlp.vim'
" precision colorscheme for the vim text editor
Plug 'altercation/vim-colors-solarized'
" lean & mean status/tabline for vim that's light as air
Plug 'vim-airline/vim-airline'
" Vim plugin for intensely orgasmic commenting
Plug 'scrooloose/nerdcommenter'
" A Vim plugin which shows a git diff in the gutter (sign column) and stages/undoes hunks
Plug 'airblade/vim-gitgutter'
" Vim motions on speed
Plug 'easymotion/vim-easymotion'
" Vim plugin for the Perl module / CLI script 'ack'
Plug 'mileszs/ack.vim'
" Vim plugin for the_silver_searcher, 'ag', a replacement for the Perl module / CLI script 'ack'
Plug 'terryma/vim-multiple-cursors'
" vim-snipmate default snippets (Previously snipmate-snippets)
Plug 'honza/vim-snippets'
" Vim Cucumber runtime files
Plug 'tpope/vim-cucumber'
" Automated tag file generation and syntax highlighting of tags in Vim
Plug 'xolox/vim-misc'
Plug 'xolox/vim-easytags'
" Vim plug for switching between companion source files (e.g. ".h" and ".cpp")
Plug 'derekwyatt/vim-fswitch'
" Emmet-vim is a vim plug-in which provides support for expanding abbreviations similar to emmet.
Plug 'mattn/emmet-vim'
" Vim plugin that allows you to visually select increasingly larger regions of text using the same key combination.
Plug 'terryma/vim-expand-region'
call plug#end()

" altercation/vim-colors-solarized
"" https://github.com/altercation/vim-colors-solarized
syntax enable
let g:solarized_termcolors=256
set background=dark
colorscheme solarized

" scrooloose/nerdtree
"" open a NERDTree automatically when vim starts up
autocmd vimenter * NERDTree | wincmd p
"" close vim if the only window left open is a NERDTree
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif " close vim if the only window left open is a NERDTree
"" open a NERDTree automatically when vim starts up if no files were specified
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 0 && !exists("s:std_in") | NERDTree | endif
"" open NERDTree automatically when vim starts up on opening a directory
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 1 && isdirectory(argv()[0]) && !exists("s:std_in") | exe 'NERDTree' argv()[0] | wincmd p | ene | endif
"" workspace dir
let g:NERDTreeChDirMode = 2
"" toggle tree
map <Leader>n :NERDTreeToggle<Enter>
"" locate current file in the tree
map <Leader>j :NERDTreeFind<Enter>
inoremap <Leader>j <Esc>:NERDTreeFind<Enter>

" xolox/vim-easytags
let g:easytags_cmd = '/usr/local/bin/ctags'

" bling/vim-airline
"" smarter tab line
let g:airline#extensions#tabline#enabled = 1
"" configure the formatting of filenames
let g:airline#extensions#tabline#fnamemod = ':t'

" kien/ctrlp.vim
"" Recent file
map <Leader>e :CtrlPBuffer<Enter>
"" The maximum number of files to scan
let g:ctrlp_max_files = 0
"" workspace dir
let g:ctrlp_working_path_mode = 'rw'
"" caching by not deleting the cache files
let g:ctrlp_clear_cache_on_exit = 0

" mileszs/ack.vim
map <Leader>f :Ack!<Space>
map <Leader>fo :Ack! --objc <cword>
map <Leader>fs :Ack! --swift <cword>
map <Leader>fp :Ack! --python <cword>
map <Leader>fj :Ack! --java <cword>

" derekwyatt/vim-fswitch
nmap <silent> <Leader>of :FSHere<cr>
nmap <silent> <Leader>ol :FSRight<cr>
nmap <silent> <Leader>oL :FSSplitRight<cr>
nmap <silent> <Leader>oh :FSLeft<cr>
nmap <silent> <Leader>oH :FSSplitLeft<cr>
nmap <silent> <Leader>ok :FSAbove<cr>
nmap <silent> <Leader>oK :FSSplitAbove<cr>
nmap <silent> <Leader>oj :FSBelow<cr>
nmap <silent> <Leader>oJ :FSSplitBelow<cr>

if has("autocmd")
autocmd FileType xml,html,c,cs,java,perl,shell,bash,cpp,python,vim,php,ruby set number
autocmd FileType xml,html vmap <C-o> <ESC>'<i<!--<ESC>o<ESC>'>o-->
autocmd FileType java,c,cpp,cs vmap <C-o> <ESC>'<o
autocmd FileType html,text,php,vim,c,java,xml,bash,shell,perl,python setlocal textwidth=100
autocmd Filetype html,xml,xsl source $VIMRUNTIME/plugin/closetag.vim
autocmd BufReadPost *
\ if line("'\"") > 0 && line("'\"") <= line("$") |
\ exe " normal g`\"" |
\ endif
endif "has("autocmd")
execute pathogen#infect()
call pathogen#helptags()
```
