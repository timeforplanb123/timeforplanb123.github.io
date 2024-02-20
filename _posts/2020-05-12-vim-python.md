---
title: vim + python
excerpt: "Для тру девелопмента на python вам нужен vim."
categories:
  - code
tags:
  - python
  - vim
toc: true
toc_label: "Getting Started"
---
**UPD 2024-02-20:** актуально для Ubuntu 22.04 с system Python 3.10
{: .notice--info}

Расскажу о своей настройке Vim для работы с Python day by day.

## Установка

### *NIX / Linux

В вашем *NIX / Linux, скорее всего уже есть vim.  
Если vim'а нет, устанавливаем:
```bash
sudo apt update
sudo apt install vim
```
Проверяем:
```bash
vim --version
```
Вывод должен быть похож на этот:
```text
VIM - Vi IMproved 8.2 (2019 Dec 12, compiled Dec 05 2023 17:58:57)
Included patches: 1-579, 1969, 580-1848, 4975, 5016, 5023, 5072, 2068, 1849-1854, 1857, 1855-1857, 1331, 1858, 1858-1859, 1873, 1860-1969, 1992, 1970-1992, 2010, 1993-2068, 2106, 2069-2106, 2108, 2107-2109, 2109-3995, 4563, 4646, 4774, 4895, 4899, 4901, 4919, 213, 1840, 1846-1847, 2110-2112, 2121
Modified by team+vim@tracker.debian.org
Compiled by team+vim@tracker.debian.org
Huge version without GUI.  Features included (+) or not (-):
+acl               +file_in_path      +mouse_urxvt       -tag_any_white
+arabic            +find_in_path      +mouse_xterm       -tcl
+autocmd           +float             +multi_byte        +termguicolors
+autochdir         +folding           +multi_lang        +terminal
-autoservername    -footer            -mzscheme          +terminfo
-balloon_eval      +fork()            +netbeans_intg     +termresponse
+balloon_eval_term +gettext           +num64             +textobjects
-browse            -hangul_input      +packages          +textprop
++builtin_terms    +iconv             +path_extra        +timers
+byte_offset       +insert_expand     -perl              +title
+channel           +ipv6              +persistent_undo   -toolbar
+cindent           +job               +popupwin          +user_commands
-clientserver      +jumplist          +postscript        +vartabs
-clipboard         +keymap            +printer           +vertsplit
+cmdline_compl     +lambda            +profile           +vim9script
+cmdline_hist      +langmap           -python            +viminfo
+cmdline_info      +libcall           +python3           +virtualedit
+comments          +linebreak         +quickfix          +visual
+conceal           +lispindent        +reltime           +visualextra
+cryptv            +listcmds          +rightleft         +vreplace
+cscope            +localmap          -ruby              +wildignore
+cursorbind        -lua               +scrollbind        +wildmenu
+cursorshape       +menu              +signs             +windows
+dialog_con        +mksession         +smartindent       +writebackup
+diff              +modify_fname      +sodium            -X11
+digraphs          +mouse             -sound             -xfontset
-dnd               -mouseshape        +spell             -xim
-ebcdic            +mouse_dec         +startuptime       -xpm
+emacs_tags        +mouse_gpm         +statusline        -xsmp
+eval              -mouse_jsbterm     -sun_workshop      -xterm_clipboard
+ex_extra          +mouse_netterm     +syntax            -xterm_save
+extra_search      +mouse_sgr         +tag_binary        
-farsi             -mouse_sysmouse    -tag_old_static  
```

Что нас здесь интересует? Поддержка python3 (python2, наверное, уже не так актуален).

Теперь проверим версию python, используемую в vim:
```bash
# открываем vim
vim
# открываем интерпретатор Python из vim
:!python3
```
```bash
Python 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```
Вероятно, дальше будет немного проще, если открыть мой [.vimrc](){:target="_blank"}.

## Плагины vim

Плагины нужны, чтобы превратить vim в IDE.

### Менеджер плагинов

Существуют разные плагин-менеджеры. Хороший и популярный вариант - [Vundle](https://github.com/VundleVim/Vundle.vim){:target="_blank"}. Скопируем в директорию для расширений vim:
```bash
git clone https://github.com/gmarik/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```
Настроим Vundle.

Вероятно, у вас уже есть `.vimrc` в домашней директории. Если нет, создадим:
```bash
touch ~/.vimrc
```
Откроем `~/.vimrc`:
```bash
vim ~/.vimrc
```
Копируем (пока, можно сделать это через контекстное меню):
```text
set nocompatible              " required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" alternatively, pass a path where Vundle should install plugins
" call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'gmarik/Vundle.vim'

" add all your plugins here (note older versions of Vundle
" used Bundle instead of Plugin)

" All of your Plugins must be added before the following line
call vundle#end()            " required
filetype plugin indent on    " required
" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
```
Теперь разберемся, как устанавливать и удалять плагины.

Например, [NERDTree](https://github.com/preservim/nerdtree){:target="_blank"}, file browsing плагин - открывает в отдельном вертикальном split'е дерево файлов и директорий.
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/nerdtree.png"><img src="{{ site.baseurl }}/assets/images/vimrc/nerdtree.png"></a>
</figure>

Достаточно добавить `Plugin 'scrooloose/nerdtree'` между строками `call vundle#begin()` и `call vundle#end()` (там, где написано `add all your plugins`) и выполнить `:PluginInstall`
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/plugin_install.png"><img src="{{ site.baseurl }}/assets/images/vimrc/plugin_install.png"></a>
</figure>

При установке появится окно установщика и `+` напротив установленного плагина.

Чтобы удалить - уберите `Plugin 'scrooloose/nerdtree'` и выполните `:PluginClean`. Выше небольшой help.

В общем, всё это справедливо для большинства vim плагинов.

Добьем вопрос с NERDTree и установим плагин [NERDTreetabs](https://github.com/jistr/vim-nerdtree-tabs){:target="_blank"} для работы в разных вкладках. По аналогии с NERDTree добавляем в ~/.vimrc:
```text
Plugin 'jistr/vim-nerdtree-tabs'
```
Устанавливаем
```text
:PluginInstall
```
Мы, все ещё, не вышли из vim.
Проверим список установленных плагинов:
```text
`:PluginList
```
Сохраним и выйдем из vim:
```text
:wq!
```
Кстати, можно выполнять команды Vundle из терминала, например, так:
```bash
vim +PluginInstall
# или
vim +PluginList
```
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/vim_plugin_list.png"><img src="{{ site.baseurl }}/assets/images/vimrc/vim_plugin_list.png"></a>
</figure>

### Key combinations

Надо, надо, ребята, потратить свое время на изучение основных команд vim. Свои основные кнопки написал в самом низу страницы.

### Split Layouts

Можно открывать файлы в вертикальном и горизонтальном сплитах, а в этих сплитах открывать новые сплиты.
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/splits.png"><img src="{{ site.baseurl }}/assets/images/vimrc/splits.png"></a>
</figure>

Настроим зоны для новых сплитов (новые вертикальные будут открываться справа, горизонтальные - внизу):
```text
set splitbelow
set splitright
```
Настроим навигацию между сплитами:
```text
" split navigations
nnoremap <C-J> <C-W><C-J>
nnoremap <C-K> <C-W><C-K>
nnoremap <C-L> <C-W><C-L>
nnoremap <C-H> <C-W><C-H>
```
```text
Ctrl+J переключиться вниз
Ctrl+K переключиться вверх
Ctrl+L переключиться вправо
Ctrl+H переключиться влево
```
nnoremap переназначает одну комбинацию клавиш на другую, при работе в Normal Mode. Например, было `<Ctrl+W><Ctrl+J>`, стало `<Ctrl+J>`. При этом `<Ctrl+W>` также будет работать.
[Подробнее](http://stackoverflow.com/questions/3776117/what-is-the-difference-between-the-remap-noremap-nnoremap-and-vnoremap-mapping){:target="_blank"}

Чтобы создать новый split в vim существуют команды:
```text
# создать вертикальный split
vsplit
# создать горизонтальный split
split
```

### Code Folding

Полезная фича - сворачивание кода. Сворачивает до ближайшего whitespace на основе отступов (foldmethod=indent). Выглядит как-то так:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/code_folding.png"><img src="{{ site.baseurl }}/assets/images/vimrc/code_folding.png"></a>
</figure>

Добавляем в наш .vimrc:
```text
" Enable folding
set foldmethod=indent
set foldlevel=99
```
По умолчанию, работает по комбинации `za`. Меняю на пробел:
```text
" Enable folding with the spacebar
nnoremap <space> za
```
Чтобы код сворачивался аккуратнее лучше установить какой-нибудь плагин, добавляем:
```text
Plugin 'tmhedberg/SimpylFold'
```
Устанавливаем этот плагин:
```text
:PluginInstall
```

### Python indentation

Для корректного code folding и соответствия PEP8 настроим отступы и длину строки:
```text
au BufRead,BufNewFile *.py,*pyw set tabstop=4
au BufRead,BufNewFile *.py,*pyw set softtabstop=4
au BufRead,BufNewFile *.py,*pyw set autoindent
au BufRead,BufNewFile *.py,*pyw set shiftwidth=4
au BufRead,BufNewFile *.py,*.pyw set expandtab
au BufRead,BufNewFile *.py,*.pyw,*.c,*.h set textwidth=88
au BufNewFile *.py,*.pyw,*.c,*.h set fileformat=unix

" for full stack development
au BufNewFile,BufRead *.js, *.html, *.css set tabstop=2
au BufNewFile,BufRead *.js, *.html, *.css set shiftwidth=2
au BufNewFile,BufRead *.js, *.html, *.css set softtabstop=2
```
Autoindent не всегда будет работать корректно, но это лечится плагином (устанавливаем через Vundle):
```text
Plugin 'vim-scripts/indentpython.vim'
```

### Flagging Unnecessary Whitespace

Лишние пробелы в коде лучше сразу удалять. Создадим флаг и подсветим красным:
```text
" Use the below highlight group when displaying bad whitespace is desired.
highlight BadWhitespace ctermbg=red guibg=red

" Make trailing whitespace be flagged as bad.
au BufRead,BufNewFile *.py,*.pyw,*.c,*.h match BadWhitespace /\s\+$/
```
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/red_whitespaces.png"><img src="{{ site.baseurl }}/assets/images/vimrc/red_whitespaces.png"></a>
</figure>

Чтобы удалить найденные пробелы можно воспользоваться заменой: 
```text
:%s/\s\+$//e
```
Можете выбрать подходящий вам вариант обработки пробелов [здесь](https://vim.fandom.com/wiki/Remove_unwanted_spaces){:target="_blank"}

### UTF-8 Support

Для работы с Python3 vim должен уметь кодировать `utf-8`. Добавим в `~/.vimrc`:
```text
set encoding=utf-8
```

### Virtualenv Support

Здесь написал про подготовку рабочего окружения и виртуальные окружения в Python - [тык](https://timeforplanb123.github.io/notes/python-environment/){:target="_blank"}

По умолчанию, vim ничего не знает об используемом виртуальном окружении. Если вы настроили виртуальные окружения, то переключаться между ними можно внутри vim, вручную, с помощью плагина [vim-virtualenv](https://github.com/jmcantrell/vim-virtualenv){:target="_blank"}. При этом vim подскажет, какое окружение активно в данный момент.

Установим через Vundle:
```text
# добавляем в ~/.vimrc
Plugin 'jmcantrell/vim-virtualenv'

# запускаем установку
:PluginInstall
```

Теперь, активируем созданное раньше виртуальное окружение:
```text
:VirtualEnvActivate <tab>
```

<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/virtualenv.png"><img src="{{ site.baseurl }}/assets/images/vimrc/virtualenv.png"></a>
</figure>

Деактивируем:
```text
:VirtualEnvDeactivate
```

### Auto-Complete

Пользуюсь автодополнением. Использование плагина [YouCompleteMe](https://github.com/ycm-core/YouCompleteMe){:target="_blank"} не принесло мне много радости. Но это субъективно. [Тут инструкция по установке](https://github.com/ycm-core/YouCompleteMe#linux-64-bit){:target="_blank"}. Здесь есть проблемы и для решения, как правило, предлагают этот [рецепт](https://github.com/ycm-core/YouCompleteMe/issues/2271){:target="_blank"}.

По душе мне пришелся [neoclide/coc.nvim](https://github.com/neoclide/coc.nvim){:target="_blank"}. Он универсален, работает как с `Jedi`, так и с `Microsoft PLS` (и то, и другое - это Language Server, еще поговорим об этом), умеет в IDE capabilities, например, покажет вам аннотации типов или `__doc__` объекта, плагин удобно настраивать, опций - огромное множество.

Установим.

Сперва, [Jedi](https://jedi.readthedocs.io/en/latest/docs/installation.html){:target="_blank"} или [Jedi](https://github.com/davidhalter/jedi){:target="_blank"}.

Jedi - это Language Server, фоновый процесс, который будет анализировать наш код. С помощью `coc.nvim`, будем просить его дополнить код, выполнить форматирование или рефакторинг.
[Здесь](https://www.vimfromscratch.com/articles/vim-and-language-server-protocol/){:target="_blank"} хорошо и кратко про `Language Server Protocol (LSP)`.

Вариантов установки много. Для примера, ставим через `pip` в нужном виртуальном окружении ([про виртуальные окружения](https://timeforplanb123.github.io/notes/python-environment/){:target="_blank"}):
```bash
# переключаемся в виртуальное окружение (у меня 3.8.4)
workon 3.8.4

# или любой другой из предложенных в документации вариантов
pip install jedi
```
Кстати, там же, в документации, есть ссылка на клиент для `Jedi` - [JEDI-VIM](https://github.com/davidhalter/jedi-vim){:target="_blank"}. Можете попробовать как альтернативу `coc.nvim`.

Теперь установим [coc](https://github.com/neoclide/coc.nvim){:target="_blank"} по [инструкции из Wiki](https://github.com/neoclide/coc.nvim/wiki/Install-coc.nvim){:target="_blank"}. Первым делом, смотрим [Requirements](https://github.com/neoclide/coc.nvim/wiki/Install-coc.nvim#requirements){:target="_blank"} и устанавливаем `Node.js` нужной версии (на текущий момент, `coc.nvim` просит `node` >= `16.18`):
- в репах обычно лежит старая версия (сейчас это 12.22.9), поэтому устанавливать лучше с оф. ресурса
- идем на [https://nodejs.org/en](https://nodejs.org/en){:target="_blank"}, смотрим доступную версию LTS
- идем на [https://nodejs.org/en/download](https://nodejs.org/en/download){:target="_blank"}, выбираем инструкцию для установки - [Installing Node.js via binary archive](https://github.com/nodejs/help/wiki/Installation){:target="_blank"}
- устанавливаем по выбранной инструкции (не дублирую сюда инструкцию)
- проверяем:
  ```bash
  node -v
  
  npm version
  
  npx -v
  ```
Теперь устанавливаем `coc.nvim` [для vim8](https://github.com/neoclide/coc.nvim/wiki/Install-coc.nvim#using-vim8s-native-package-manager){:target="_blank"}:
```bash
mkdir -p ~/.vim/pack/coc/start
cd ~/.vim/pack/coc/start
git clone --branch release https://github.com/neoclide/coc.nvim.git --depth=1
vim -c "helptags coc.nvim/doc/ | q"
```
Находясь в `~/.vimrc`, с помощью встроенного менеджера, установим `coc-python`:
```text
:CocInstall coc-python
```
Откроем какой-нибудь python-скрипт, и, поскольку мы уже в нужном виртуальном окружении(настроили раньше, в [Virtualenv Support](https://timeforplanb123.github.io/code/vim-python/#virtualenv-support)), переключим интерпретатор на работу с `Jedi` в этом же окружении. Делается это снова встроенным менеджером:
```text
:CocCommand
# здесь мы должны выбрать то виртуальное окружение, куда раньше установили Jedi
# опция будет постоянной и при работе в другом виртуальном окружении, нужно переключить coc.nvim

python.setInterpreter
```
В появившемся окне выбираем виртуальное окружение, куда ранее установили `Jedi`. Вот в картинках, до момента выбора окружения:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/coccommand.png"><img src="{{ site.baseurl }}/assets/images/vimrc/coccommand.png"></a>
    <a href="{{ site.baseurl }}/assets/images/vimrc/setinterpreter.png"><img src="{{ site.baseurl }}/assets/images/vimrc/setinterpreter.png"></a>
</figure>

С помощью `CocCommand` можно настраивать много всего. Фактически, мы правим конфигурационный файл через интерфейс встроенного менеджера. Посмотрим на интерфейс на примере функции `print`:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/completion.png"><img src="{{ site.baseurl }}/assets/images/vimrc/completion.png"></a>
</figure>

Если `coc.nvim` нужно отключить:
```text
:CocDisable
```
Включаем обратно:
```text
:CocEnable
```
Теперь, добавим полезного в наш `~/.vimrc` для работы с `coc`:
```text
let mapleader = "," 

nnoremap <silent> K :call <SID>show_documentation()<CR>
function! s:show_documentation()
  if (index(['vim','help'], &filetype) >= 0)
    execute 'h '.expand('<cword>')
  else
    call CocActionAsync('doHover')
  endif
endfunction
```
Функция включает возможность отображения документации по `K` кнопке. Смотрим, как это выглядит (ставим курсор на `print`, жмем `K`):
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/docstring.png"><img src="{{ site.baseurl }}/assets/images/vimrc/docstring.png"></a>
</figure>

Добавим возможность просмотра аннотаций типов в python для выбранной переменной. Добавляем в `~/.vimrc`:
```text
nmap <silent> gd :call CocActionAsync('jumpDefinition', 'vsplit')<CR>
```
Для того, чтобы `vsplit` работал корректно, нужно добавить `"coc.preferences.jumpCommand": "vsplit"` в `coc-settings.json` (`coc-settings.json` открывается командой `:CocConfig` внутри vim).

Для проверки выбираем любую переменную в python-коде, для которой определена аннотация, жмем `gd`, в вертикальном сплите откроется строка, где определена аннотация для этой переменной. К сожалению, если сделать тоже самое с выбранной функцией или классом, можно увидеть определение выбранных функции или класса (расположение и код), но только в горизонтальм сплите. Это связано с каналом вывода для `coc.vim` и `vim`. Я не разобрался, но копать можно отсюда - [Using output channel](https://github.com/neoclide/coc.nvim/wiki/Debug-language-server#using-output-channel){:target="_blank"}, [How to jump to definition in new or existing tab #318](https://github.com/neoclide/coc.nvim/issues/318)

У проекта отличная [Wiki](https://github.com/neoclide/coc.nvim/wiki){:target="_blank"}, если вдруг.

### File Browsing

Плагины [nerdtree](https://github.com/preservim/nerdtree){:target="_blank"} и [vim-nerdtree-tabs](https://github.com/jistr/vim-nerdtree-tabs){:target="_blank"} мы уже установили.

Давайте добавим в `~/.vimrc` строку:
```text
let NERDTreeIgnore=['\.pyc$', '\~$']
```
NERDTree будет игнорировать `.pyc`

Если у вас есть вопрос по работе NERDTree, скорее всего он уже решен - [F.A.Q.](https://github.com/preservim/nerdtree/wiki/F.A.Q.){:target="_blank"}

Команды для работы с vim-nerdtree-tabs описаны в [Commands and Mappings](https://github.com/jistr/vim-nerdtree-tabs){:target="_blank"}

Добавим в `~/.vimrc` возможность открытия NERDTree по `<F3>`:
```text
" map NERDTree on F3
map <F3> :NERDTreeToggle<CR>
```

### Syntax Checking/Highlighting

Добавляем в `~/.vimrc` плагин для проверки синтаксиса:
```text
Plugin 'vim-syntastic/syntastic'
```
Добавляем линтер для PEP8:
```text
Plugin 'nvie/vim-flake8'
```
Включаем подсветку синтаксиса:
```text
let python_highlight_all=1
syntax on
```

### Поддержка markdown

Ну, а почему нет? Подсветка синтаксиса и превью - это все, что нужно.

Смотрим топ плагинов [здесь](https://vimawesome.com/?q=markdown){:target="_blank"}.

Хорошие - [Vim Markdown](https://github.com/preservim/vim-markdown){:target="_blank"} и [vim-instant-markdown](https://github.com/instant-markdown/vim-instant-markdown){:target="_blank"} как превью. Устанавливаем уже знакомым способом, через `Vundle`:
```text
vim ~/.vimrc

# добавляем
Plugin 'instant-markdown/vim-instant-markdown'
Plugin 'godlygeek/tabular' "for work with markdown plugin (Ctrl-p) - https://github.com/godlygeek/tabular
Plugin 'preservim/vim-markdown' "best markdown plugin - https://github.com/instant-markdown/vim-instant-markdown

# запускаем Vundle
:PluginInstall
```
Для `vim-instant-markdown` необходимо дополнительно [установить nodejs mini-server](https://github.com/instant-markdown/vim-instant-markdown?tab=readme-ov-file#installation){:target="_blank"}. Поскольку nodejs уже установлен, выполняем:
```bash
npm -g install instant-markdown-d
```
Добавим для `vim-instant-markdown` доступные настройки в `~/.vimrc` (оставляю закомментированными, т.к. использую дефолт):
```text
filetype plugin on
"Uncomment to override defaults:
"let g:instant_markdown_slow = 1
"let g:instant_markdown_autostart = 0
"let g:instant_markdown_open_to_the_world = 1
"let g:instant_markdown_allow_unsafe_content = 1
"let g:instant_markdown_allow_external_content = 0
"let g:instant_markdown_mathjax = 1
"let g:instant_markdown_logfile = '/tmp/instant_markdown.log'
"let g:instant_markdown_autoscroll = 0
"let g:instant_markdown_port = 8888
"let g:instant_markdown_python = 1
```
Добавим подсветку YAML, JSON для `vim-markdown`, разрешим открывать ссылки на `.md` в новых вкладках:
```text
" vim markdown plugin options - https://github.com/plasticboy/vim-markdown
let g:vim_markdown_frontmatter = 1
let g:vim_markdown_json_frontmatter = 1
let g:vim_markdown_edit_url_in = 'tab'
```
Для удобства настроим отсупы аналогично Python indentation. Мы уже делали это ранее, теперь только добавим `*md` и `*markdown`:
```text
au BufRead,BufNewFile *.py,*pyw,*md,*markdown set tabstop=4
au BufRead,BufNewFile *.py,*pyw,*md,*markdown set softtabstop=4
au BufRead,BufNewFile *.py,*pyw,*md,*markdown set autoindent
au BufRead,BufNewFile *.py,*pyw,*md,*markdown set shiftwidth=4
au BufRead,BufNewFile *.py,*.pyw,*md,*markdown set expandtab
```
Например, теперь, помимо подсветки, можно открыть в отдельном сплите все заголовки вашего markdown-файла и переключаться между ними:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/vimmarkdowncommands.png"><img src="{{ site.baseurl }}/assets/images/vimrc/vimmarkdowncommands.png"></a>
</figure>

А превью будет сразу открываться как новая вкладка в браузере:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/preview.png"><img src="{{ site.baseurl }}/assets/images/vimrc/preview.png"></a>
</figure>

### Color Schemes Switching

У меня много схем оформления vim и возможность переключения между ними по `<F8>`.

Для начала скопируем в `~/.vim/colors/` схемы оформления. Возьмем их, например, [здесь](https://github.com/rainglow/vim){:target="_blank"} и [здесь](https://github.com/rafi/awesome-vim-colorschemes){:target="_blank"} (достаточно скопировать только содержимое директории colors).

В качестве постоянной схемы я использую `dogrun`. В `~/.vimrc` добавить схему как постоянную, можно так:
```text
colorscheme dogrun
```
Добавим схемы оформления в vim:
```text
:SetColors all
```
Добавим возможность переключения по `<F8>`, скопировав [этот скрипт](https://vim.fandom.com/wiki/Switch_color_schemes){:target="_blank"} в `~/.vim/plugin/setcolors.vim`

Чтобы отобразить текущую схему, можно использовать:
```text
:SetColors
```

### Super Searching

Расширенные возможности поиска в vim будут добавлены вместе с плагином [ctrlP](https://github.com/kien/ctrlp.vim){:target="_blank"}. Устанавливаем обычно, через Vundle:
```text
# добавляем в `/.vimrc
Plugin 'kien/ctrlp.vim

# устанавливаем
:PluginInstall
```

### Status bar with vim-airline

Полезнейшая в работе вещь - статус бар.
Есть [powerline](https://github.com/powerline/powerline){:target="_blank"}, но я использую [vim-airline](Plugin 'vim-airline/vim-airline-themes'){:target="_blank"}

Добавляем в `~/.vimrc` и устанавливаем через Vundle:
```text
Plugin 'vim-airline/vim-airline'
Plugin 'vim-airline/vim-airline-themes'
```
Настраиваем:
```text
" air-line settings 
" enable tab line with vim-airline plugin
let g:airline#extensions#tabline#enabled = 1
let g:airline_skip_empty_sections = 1
let g:airline_theme='minimalist'
let g:airline_section_y = '%{virtualenv#statusline()}'
```
Теперь, vim будет выглядеть так:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/vim_airline.png"><img src="{{ site.baseurl }}/assets/images/vimrc/vim_airline.png"></a>
</figure>

vim-airline интегрируется со многими плагинами, например, с `ctrlP`, который мы уже установили.

В vim-airline можно добавить, например, наименование используемого виртуального окружения, что мы и сделали.

[Подробнее](https://github.com/vim-airline/vim-airline){:target="_blank"}

### Git Integration

Как по мне, лучший плагин для интеграции vim с git - это [fugitive.vim](https://github.com/tpope/vim-fugitive){:target="_blank"}, хотя бы, по той причине, что он дублирует уже привычный набор команд git и не требует особого изучения. Устанавливается, как обычно, через Vundle:
```text
# добавляем в `/.vimrc
Plugin 'tpope/vim-fugitive'

# устанавливаем
:PluginInstall
```
Добавим в статус бар:
```text
" to add fugitive plugin (for git) in statusline
let g:airline_section_b = '%{FugitiveStatusline()}'
```
Для работы с git есть плагин [vimagit](https://github.com/jreybert/vimagit){:target="_blank"}, но мне он кажется менее удобным.

Для отображения статуса git-репозитория в NERDTree попробуйте [этот плагин](https://github.com/Xuyuanp/nerdtree-git-plugin){:target="_blank"}

Мне не очень понравилось работать с git из под vim, поэтому в своем `~/.vimrc` я отключил vim-fugitive плагин, но оставил nerdtree-git-plugin.

### Line Numbering

Нумерация строк будет отображаться в статус баре, который мы установили ранее. Это удобно. Но, если вы хотите отображать нумерацию строк в vim, добавьте в `~/.vimrc`:
```text
set nu
```

### PasteToggle

Иногда вам будет полезна опция `:set paste`. Используется до вставки скопированного кода "как есть", без autoindent. Актуально при работе в insert mode. Добавляем на <F2> в `~/.vimrc`:
```text
set pastetoggle=<F2>
```
Копируем код, переходим в insert mode, включаем  pastetoggle (F2), вставляем код, отключаем pastetoggle(F2). [Подробнее](https://vim.fandom.com/wiki/Toggle_auto-indenting_for_code_paste){:target="_blank"}

### Поддержка black и запуск python

[Black](https://github.com/psf/black){:target="_blank"} - популярный форматтер python-кода. Устанавливаем глобально, через pip:
```bash
pip install black
```
На этой же [странице](https://github.com/psf/black){:target="_blank"} есть плагин для vim.

Применять black достаточно просто, поэтому отдельным плагином я не пользуюсь. Основные команды black:
```bash
black script.py (форматировать файл script.py)
black script.py -l 120 (форматировать с длиной строки 120. По умолчанию 88)
black --diff script.py (посмотреть изменения в формате, но не форматировать)
black . (форматировать все файлы в текущей директории)
```
А в `~/.vimrc` добавим hotkey на `<F9>` для запуска black как стороннего инструмента:
```text
" manual black code reformatting
nnoremap <F9> :w<CR>:!clear;black %<CR>
```
Теперь добавим возможность сохранения и запуска  python-интерпретатора на `<F5>`:
```text
" save and run current python code
nnoremap <F5> ::w!<CR>:!clear;python %<CR>
```

### Вкладки в vim

Добавим клавиши для переключения вкладок в vim:
```text
" Ctrl-Left or Ctrl-Right to go to the previous or next tabs
nnoremap <C-Left> :tabprevious<CR>
nnoremap <C-Right> :tabnext<CR>
" Alt-Left or Alt-Right to move the current tab to the left or right
nnoremap <silent> <C-Down> :execute 'silent! tabmove ' . (tabpagenr()-2)<CR>
nnoremap <silent> <C-Up> :execute 'silent! tabmove ' . (tabpagenr()+1)<CR>
```
Ctrl+Left - переключить на предыдущую вкладку
Ctrl+Right - переключить на следуюущую вкладку
Alt+Left - переместить вкладку назад
Alt+Right - переместить вкладку вперед

Вкладки в vim:
<figure>
    <a href="{{ site.baseurl }}/assets/images/vimrc/tabs.png"><img src="{{ site.baseurl }}/assets/images/vimrc/tabs.png"></a>
</figure>

### Switching Buffers

[Про работу с buffers в vim](https://vim.fandom.com/wiki/Using_tab_pages){:target="_blank"}

Для переключения добавим hotkey на `<F7>`:
```text
" switching to another buffer manually - https://vim.fandom.com/wiki/Using_tab_pages
" :help switchbuf
set switchbuf=usetab
nnoremap <F7> :sbnext<CR>
nnoremap <S-F7> :sbprevious<CR>
```

### History

Добавим полезную настройку - разрешим хранить историю после выхода из файла:
```text
" Maintain undo history between sessions
set undofile
set undodir=~/.vim/undodir
```


## Hotkeys

`i` - insert - режим ввода/редактирования

`esc` - выйти из режима редактирования/визуального режима (можно применять нужное кол-во раз)

`dw` - удалить слово

`dd` - удалить строку

`d$` - удалить всё от текущего месторасположения курсора до конца строки

`d^` - удалить всё от текущего месторасположения курсора до начала строки

`:15 dgg` - прыгнуть на 15 строку и удалить всё, начиная с 15 строки, до начала файла (dG - до конца файла)

`dt'` - удалить все символы в строке от текущего месторасположения до символа одинарной кавычки (можно использовать любой символ)

`5dd` - удалить 5 строк

`5dw` - удалить 5 слов

Символ удаления `d` можно комбинировать с поиском. Например, чтобы удалить все от курсора до конкретного слова, жмем `d`, открываем поиск с помощью `/`, пишем слово, до которого удаляем.

`yy` - скопировать строку (никто не отменял копирование через контестное меню)

`yw` - скопировать слово

`10yy` - скопировать 10 строк

`p` - вставить после курсора (не всегда удобно)

`P` - вставить до курсора

`:q` - выйти из файла

`:q!` - выйти из файла жестко, без сохранения изменений

`:w` - сохранить файл

`:wq` - сохранить файл и выйти

`ctrl+V` - перейти в визуальный режим

`u` - отменить действие (undo)

`ctrl+R` - return или undo undo (повторить действие)

`hjkl` - передвижение по vim

`^` - начало строки (вернуться к первому не пустому символу в строке)

`0` - вернуться в самое начало строки

`$` - вернуться в конец строки

`A` - вернуться в конец строки и открыть режим редактирования

`I` - вернуться в начало строки и открыть режим редактирования

`o` - прыгнуть на следуюущую строку и перейти в режим редактирования

`w` - передвижение на одно слово вперед

`W` - передвижение от пробела к пробелу (через слово)

`b` - передвижение назад, от слова к слову

`B` - передвижение назад, от пробела к пробелу

`gg` - прыгнуть в начало файла

`G` - прыгнуть в конец файла

`30G` - прыгнуть на нужную строку (для gg аналогично)

`:55` - передвинуть курсор на 55 строку

`ctrl+D` - листать постранично вниз

`ctrl+U` - листать постранично вверх

`zt` - при нахождении на строке, которая расположена в нижней части терминала, поднимаем эту строку на самый верх

`zz` - аналогично предыдущему, но поднимаем строку на середину

`/` - поиск (здесь лучше сразу обратить внимание на инкрементальный режим, т.е. поиск в реальном времени). В поиске есть история - работает нажатием вверх или вниз (по аналогии с :).
Для передвижения по результатам поиска жмем `n` и `N`

`?` - поиск в обратную сторону, по аналогии с `/`

`.` - повторить предыдущую команду

`:s/чтозаменить/начтозаменить` - замена в строке

`:%s/чтозаменить/начтозаменить/` - замена во всем файле (если слово в строке встречается дважды, добавьте g, либо установите set gdefault

`>>` - сдвинуть строку вправо

`<<` - сдвинуть строку влево

`12>` - сдвинуть 12 строк вправо (аналогично влево)

`:vs имя файла` - вертикальный сплит

`:sp` - горизонтальный сплит

`:vertical resize30%` - изменить размер текущего сплита

`:resize` - изменить размер для горизонтального сплита

`za` - свернуть код

`vim  -p file1 file2 file3` - открыть несколько файлов в разных вкладках

`:tabedit имя файла` - открыть файл в новой вкладке

`:tabn (можно с номером вкладки`) - перейти на следующую вкладку

`:tabp` - перейти на предыдущую вкладку

`:tabc` - закрыть вкладку

`:tabfirst` - перейти на первую вкладку

`:tablast` - перейти на последнюю вкладку

`:tabs` - открыть список доступных вкладок

`:tabl` - прыгнуть на последнюю открытую вкладку

`:tab split` - скопировать содержимое текущей вкладки в новую вкладку и перейти на неё

`:tabonly` - закрыть все вкладки, кроме текущей

`:tab ball` или `:tabo` - показать все буферы во вкладках

`:ls` - посмотреть буферы

`q:` - открыть историю буферов

`:qa` - закрыть всё

`!ls` - выполнить shell команду из vim

`!python file.py` - запустить python код (по аналогии с !ls)


Чтобы  добавить вывод команды, запущенной из под vim, в файл, выполняем:
```text
vim new_file.txt
:read !python file.py - вывод file.py попадет в new_file.txt
:read !ls -ls - вывод команды попадет в файл new_file.txt
```

Комментируем несколько строк в визуальном режиме:
- `ctrl+V` - переходим в визуальный режим (`shift+V` - выделить всю строку)
- выделяем нужное кол-во строк с помощью указателей (`jk` или `вверх/вниз`)
- `shift+I` - переходим в режим вставки, пишем символ решетки `#` (а, например, если нужно удалить по символу в каждой из выделенных строк, жмем `x`)
- `esc` - возвращаемся в обычный режим (видим результат)


`vim -S session.vim` - открыть сохраненную сессию


## Resources

1. [Основной источник этой статьи](https://realpython.com/vim-and-python-a-match-made-in-heaven/#vim-extensions){:target="_blank"}
2. [Второй источник этой статьи](https://www.vimfromscratch.com/articles/vim-for-python/){:target="_blank"}
3. [Vim and Language Server Protocol](https://www.vimfromscratch.com/articles/vim-and-language-server-protocol/){:target="_blank"}
4. [Лекции по основам vim](https://www.youtube.com/playlist?list=PLah0HUih_ZRkiQXDuElo_JW9OfmbEXRpj){:target="_blank"}
5. [Mapping keys](https://vim.fandom.com/wiki/Mapping_keys_in_Vim_-_Tutorial_(Part_2)){:target="_blank"}
6. [Using tab pages](https://vim.fandom.com/wiki/Using_tab_pages){:target="_blank"}
7. [Отличный сборник плагинов для vim](https://vimawesome.com/){:target="_blank"}
