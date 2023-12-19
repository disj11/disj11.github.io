---
title: "Windows Initial Setup"
description: "Windows 설치 후 초기 설정해야 하는 것들을 기록"
date: 2023-01-30T09:10:13+09:00
tags: ["settings"]
---

# Windows 초기 설정

Windows 최초 설치 후 설정 정리

## 날개셋 한글 입력기 설치

세벌식 유저이므로 [날개셋 한글 입력기](http://moogi.new21.org/ngs_download.htm)를 다운받는다.

vim 유저이므로 ESC 누를 시 영문으로 전환하는 기능을 설정한다. 설정 방법은 아래 이미지를 참고한다.

![ESC key mappings](/images/notes/esc-key-mappings.png)

## PowerToys

Microsoft Store 에서 Microsoft PowerToys 를 찾아 설치한다.

## Scoop

[scoop](https://scoop.sh/) 을 다운 받는다.

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser # Optional: Needed to run a remote script the first time
irm get.scoop.sh | iex
```

## Git

Git 을 다운 받는다.

```powershell
scoop install git
```

## VIM

[gvim](https://www.vim.org/download.php) 을 다운받는다.

설치 시 Create .bat files 를 체크한다. 이 옵션을 체크하면 명령 프롬프트에서 vim 명령 사용이 가능해진다.

![Vim Install Setup](/images/notes/vim-create-bat-files.png)

[vim-plug](https://github.com/junegunn/vim-plug) 를 설치한다. powershell 을 실행하여 다음 명령어를 입력한다.

```powershell
iwr -useb https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim |`
    ni $HOME/vimfiles/autoload/plug.vim -Force
```

_vimrc 파일을 작성한다. `C:\Users\사용자계정` 위치에 _vimrc 파일을 생성한 뒤 다음 내용을 입력한다.

```
let mapleader=" "

set encoding=UTF-8

set number
set showcmd
set incsearch
set ignorecase
set scrolloff=5
set clipboard=unnamed

set smartindent
set expandtab
set tabstop=4
set shiftwidth=4
set softtabstop=4

map <Home> ^
map <End> $
nnoremap <leader>ca ggVG

" vim-plug
call plug#begin()
    Plug 'ryanoasis/vim-devicons'
    Plug 'joshdick/onedark.vim'
    Plug 'vim-airline/vim-airline'
    Plug 'vim-airline/vim-airline-themes'

    Plug 'terryma/vim-multiple-cursors'
    Plug 'preservim/nerdtree'
    Plug 'easymotion/vim-easymotion'
    Plug 'tpope/vim-commentary'
    Plug 'tpope/vim-surround'
call plug#end()

syntax on
colorscheme onedark
let g:airline_theme='onedark'

let NERDTreeMapOpenVSplit='v'
let NERDTreeMapPreviewVSplit='gv'
let NERDTreeMapOpenSplit='s'
let NERDTreeMapPreviewSplit='gs'

nnoremap <leader>nt :NERDTreeToggle<CR>
nnoremap <leader>nc :NERDTreeFind<CR>
nnoremap <leader>nf :NERDTreeFocus<CR>

nmap <leader>/ <Plug>(easymotion-bd-fn)
nmap <leader>; <Plug>(easymotion-next)
nmap <leader>, <Plug>(easymotion-prev)

if has('gui_running')
    set guifont=JetBrainsMono\ Nerd\ Font\ Mono:h13
endif

" intellij
if has('ide')
    set ideajoin
    set quickscope

    nmap <leader>* <Action>(FindUsages)
    nmap <leader>o <Action>(GotoFile)
    nmap <leader>f <Action>(FindInPath)
    nmap <leader>t <Action>(HideAllWindows)

    nmap <leader>gp <Action>(Vcs.UpdateProject)
    nmap <leader>gb <Action>(Git.Branches)

    nmap gd <Action>(GotoDeclaration)
    nmap gD <Action>(GotoImplementation)
    nmap ]m <Action>(MethodDown)
    nmap [m <Action>(MethodUp)

    " navigation
    nmap <leader>nr <Action>(RecentFiles)
    nmap <leader>nl <Action>(RecentLocations)

    " code
    nmap <leader>cd <Action>(Debug)
    nmap <leader>cr <Action>(RunAnything)
    nmap == <Action>(ReformatCode)
    nmap <leader>co <Action>(OptimizeImports)

    nmap <leader>gg <Action>(Generate)
    nmap <leader>gc <Action>(GenerateConstructor)
    nmap <leader>go <Action>(OverrideMethods)
    nmap <leader>gi <Action>(ImplementMethods)
    nmap <leader>cn <Action>(NewElementSamePlace)

    " show
    nmap <leader>sd <Action>(QuickJavaDoc)
    nmap <leader>sp <Action>(ParameterInfo)
    nmap <leader>ss <Action>(FileStructurePopup)

    " refactor
    vmap <leader>rm <Action>(ExtractMethod)
    nmap <leader>rn <Action>(RenameElement)

    " vim-multiple-cursors
    nmap <C-n> <Plug>NextWholeOccurrence
    xmap <C-n> <Plug>NextWholeOccurrence
    nmap g<C-n> <Plug>NextOccurrence
    xmap g<C-n> <Plug>NextOccurrence
    nmap <C-x> <Plug>SkipOccurrence
    xmap <C-x> <Plug>SkipOccurrence
    nmap <C-p> <Plug>RemoveOccurrence
    xmap <C-p> <Plug>RemoveOccurrence
    nmap <S-C-n> <Plug>AllWholeOccurrences
    xmap <S-C-n> <Plug>AllWholeOccurrences
    nmap g<S-C-n> <Plug>AllOccurrences
    xmap g<S-C-n> <Plug>AllOccurrences
endif
```

vim 플러그인을 설치한다. powershell 을 **관리자 권한**으로 실행한다.  `vim` 명령을 입력 후 vim 으로 진입한다. `:PlugInstall` 명령어를 통해 플러그인을 설치한다.

[Nerd Fonts](https://www.nerdfonts.com/font-downloads) 에서 JetBrainsMono Nerd Font 를 설치한다. 압축 해제하면 여러 개의 파일이 나오는데, JetBrains Mono Regular Nerd Font Complete Mono Windows Compatible 만 설치해도 된다.

## SDKMAN

자바 버전 관리를 편하게 하기 위해 [sdkman](https://sdkman.io/) 을 설치한다.

```powershell
scoop bucket add palindrom615 https://github.com/palindrom615/scoop-bucket
scoop install sdkman
```

## JDK

sdkman 을 사용하여 설치하고 싶은 자바를 선택하여 설치한다. `sdk list java` 명령어로 자바 버전을 확인할 수 있다.

## IntelliJ

[JetBrains Toolbox App](https://www.jetbrains.com/toolbox-app/) 을 설치한다. 이후 ToolBox App 에서 Intellij IDEA Ultimate 를 설치한다.

IntelliJ 설치가 완료되면 아래의 플러그인을 설치한다.

* [IdeaVim](https://plugins.jetbrains.com/plugin/164-ideavim)
* [IdeaVim-EasyMotion](https://plugins.jetbrains.com/plugin/13360-ideavim-easymotion)
* [IdeaVim-Quickscope](https://plugins.jetbrains.com/plugin/19417-ideavim-quickscope)

이외에 필요한 플러그인이 있다면, 추가적으로 설치한다.  ([Kotest](https://plugins.jetbrains.com/plugin/14080-kotest) 플러그인 등)

이후 vim 과 동일한 키 설정을 사용하기 하기 위해 `.ideavimrc` 에 다음 내용을 추가한다.

```powershell
source ~/_vimrc 
```

개발 시 JDK 가 필요하다면 SDK 에 JDK 설치 경로를 추가한다. SDKMAN 으로 설치한 JDK 는 `~/.sdkman` 위치에서 찾을 수 있다.

## Etc.

### Hugo

개인 블로그 개발 환경을 위해 hugo 설치

```powershell
scoop install hugo-extended
```
