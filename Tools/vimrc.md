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
call plug#begin('~/.vim/plugged')

" window
Plug 'scrooloose/nerdtree'
Plug 'majutsushi/tagbar'
Plug 'vim-airline/vim-airline'
Plug 'xolox/vim-misc'

" search
Plug 'junegunn/fzf'
Plug 'junegunn/fzf.vim'
Plug 'easymotion/vim-easymotion'

" edit
Plug 'terryma/vim-multiple-cursors'
Plug 'skwp/greplace.vim'
Plug 'arthurxavierx/vim-caser'
Plug 'scrooloose/nerdcommenter'

" git
Plug 'tpope/vim-fugitive'
Plug 'airblade/vim-gitgutter'
Plug 'junegunn/gv.vim'

" color
Plug 'flazz/vim-colorschemes'

" complete
Plug 'raimondi/delimitmate'
Plug 'vim-scripts/AutoComplPop'
Plug 'ervandew/supertab'
Plug 'honza/vim-snippets'
Plug 'sirver/ultisnips'
Plug 'davidhalter/jedi-vim'

" language
Plug 'sheerun/vim-polyglot'
Plug 'moll/vim-node'
Plug 'kannokanno/previm'
Plug 'othree/javascript-libraries-syntax.vim'
Plug 'ap/vim-css-color'
Plug 'hail2u/vim-css3-syntax'
Plug 'vim-scripts/a.vim'

" tools
Plug 'w0rp/ale'

call plug#end()

" global 
filetype on
syntax on 

set nu
set nowrap
set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936
set termencoding=utf-8
set encoding=utf-8
set bg=dark
set hlsearch
set nocompatible
set backspace=indent,eol,start
set smartindent
""" set ignorecase

"" color
colorscheme gruvbox
""" colorscheme molokai
""" colorscheme wombat
""" colorscheme solarized
""" colorscheme ir_black
""" colorscheme atom

"" highligt search if colorscheme does not well such as 'ir_black'
""" hi Search guibg=peru guifg=wheat

"" filetype extension 
"" java
au BufNewFile,BufRead *.java,*.jav,*.aidl setf java

"" quick edit vimrc
map <Leader><F2> :e ~/.vimrc<CR>

"" chagne buffer
map <Leader>[ :bp<Enter>
map <Leader>] :bn<Enter>

"" Disable swap
set noswapfile

"" enable clipboard 
:inoremap <C-v> <ESC>"+pa
:vnoremap <C-c> "+y
:vnoremap<C-d> "+d

"" move
nnoremap mb ^
nnoremap me $
vnoremap mb ^
vnoremap me $

"" Alt + * move line
"" http://vim.wikia.com/wiki/Moving_lines_up_or_down
nnoremap ∆ :m .+1<CR>==
nnoremap ˚ :m .-2<CR>==
inoremap ∆ <Esc>:m .+1<CR>==gi
inoremap ˚ <Esc>:m .-2<CR>==gi
vnoremap ∆ :m '>+1<CR>gv=gv
vnoremap ˚ :m '<-2<CR>gv=gv

"" tab spaces
set tabstop=4
set expandtab
set shiftwidth=4

"" font
if has('macunix')
    set guifont=Monaco:h12
elseif has('unix')
    set guifont="Ubuntu Mono" 12
endif

"" resize window
map <Leader>= :vertical resize +10<Enter>
map <Leader>- :vertical resize -10<Enter>
map <Leader>. :resize +10<Enter>
map <Leader>, :resize -10<Enter>

"" hightline current line and column
au WinLeave * set nocursorline nocursorcolumn
au WinEnter * set cursorline cursorcolumn
set cursorline cursorcolumn

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
"" toggle tree
map <Leader>w :NERDTreeToggle<Enter>
"" locate current file in the tree
nnoremap <Leader>j :NERDTreeFind<Enter>
inoremap <Leader>j <Esc>:NERDTreeFind<Enter>
""" autocmd BufWinEnter * NERDTreeFind
"" folder icon
let g:NERDTreeDirArrowExpandable = '▸'
let g:NERDTreeDirArrowCollapsible = '▾'

"" search dir
let g:NERDTreeChDirMode = 2
"" startup cursor in editing area
autocmd VimEnter * NERDTree | wincmd p

" vim-fugitive'
nnoremap <Leader>gs :Gstatus<CR>
nnoremap <Leader>gc :Gcommit<CR>
nnoremap <Leader>gb :Gblame<CR>
nnoremap <Leader>gm :Gmove<CR>
nnoremap <Leader>gd :Gdelete<CR>

" fzf
set rtp+=/usr/local/opt/fzf

nnoremap <Leader>ff :Files<CR>
nnoremap <Leader>fg :GFiles<CR>
nnoremap <Leader>fb :Buffers<CR>
nnoremap <Leader>fe :History<CR>
nnoremap <Leader>ft :Tags<CR>
nnoremap <Leader>fc :History:<CR>
nnoremap <Leader>fa :Ag<Space>
nnoremap <Leader>fs :Filetypes<CR>
nnoremap <Leader>fd :Ag <C-R><C-W><CR>

" ctags 
map <Leader>rr :!ctags -R --exclude=.git --exclude=node_modules --exclude=log *<Enter>

" bling/vim-airline
"" smarter tab line
let g:airline#extensions#tabline#enabled = 1
"" configure the formatting of filenames
let g:airline#extensions#tabline#fnamemod = ':t'

" pangloss/vim-javascript'
let g:javascript_plugin_jsdoc = 1

" terminal
set splitbelow
map <Leader>` :ter ++rows=11<CR>
tnoremap <F3> <C-\><C-n>

" tagbar
nnoremap <Leader><F12> :TagbarOpenAutoClose<CR>

" previm
if has('macunix')
    let g:previm_open_cmd = 'open -a Google\ Chrome'
elseif has('unix')
    let g:previm_open_cmd = 'open -a Firefox'
endif

map <Leader>p :PrevimOpen<CR>
""" hide header to print html page
let g:previm_show_header = 0

" supertab
let g:SuperTabMappingForward = '<s-tab>'
let g:SuperTabMappingBackward = '<tab>'

" skwp/greplace
set grepprg=ag
let g:grep_cmd_opts = '--line-numbers --noheading'
""" 1. Use :Gsearch to get a buffer window of your search results
""" 2. then you can make the replacements inside the buffer window using traditional tools (%s/foo/bar/)
""" 3. Invoke :Greplace to make your changes across all files. It will ask you interatively y/n/a - you can hit 'a' to do all.
""" 4. Save changes to all files with :wall (write all)
nnoremap <Leader>G :Gsearch<Space>

" othree/javascript-libraries-syntax.vim
let g:used_javascript_libs = 'jquery,underscore,backbone,react,vue'

" sirver/ultisnips
""" let g:UltiSnipsUsePythonVersion = 3

" Shifting blocks visually
nnoremap > >>
nnoremap < << 
vnoremap > >gv
vnoremap < <gv

" ale
"" keep the sign gutter open
let g:ale_sign_column_always = 1
let g:ale_sign_error = '>>'
let g:ale_sign_warning = '--'

"" show errors or warnings in my statusline
let g:airline#extensions#ale#enabled = 1

"" use quickfix list instead of the loclist
let g:ale_set_loclist = 0
let g:ale_set_quickfix = 1

"" only enable these linters
"" let g:ale_linters = {
"" \    'javascript': ['eslint']
"" \}

"" Fix files with prettier, and then ESLint.
let b:ale_fixers = ['prettier', 'eslint']
let g:ale_fix_on_save = 1

nmap <silent> <C-k> <Plug>(ale_previous_wrap)
nmap <silent> <C-j> <Plug>(ale_next_wrap)

"" run lint only on saving a file
let g:ale_lint_on_text_changed = 'never'
"" dont run lint on opening a file
"" let g:ale_lint_on_enter = 0
```
