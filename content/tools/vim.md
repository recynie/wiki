---
title: Vim编辑器教程
date: 2026-04-11
description: Vim编辑器从入门到精通
tags:
  - vim
  - editor
  - tools
draft: false
permalink:
---

## 概述

Vim（Vi IMproved）是从 vi 发展而来的文本编辑器，以其高效性和可定制性著称。Vim 的核心设计理念是**模态编辑**：在不同模式下，同一个按键有不同的功能。[^1]

### 为什么学习 Vim

- **高效性**：通过组合操作符和动作，可以用最少的按键完成复杂编辑
- **跨平台**：几乎所有 Linux/Unix 系统都预装了 Vim
- **可扩展**：通过插件和脚本可以无限扩展功能
- **远程工作**：在 SSH 远程环境中，Vim 是最可靠的编辑器

### 启动 Vim

```bash
# 直接打开文件
vim filename

# 打开多个文件
vim file1 file2 file3

# 进入交互式教程
vimtutor
```

---

## Vim 模式

Vim 有多种模式，理解模式是掌握 Vim 的关键。

### 模式概览

| 模式 | 用途 | 进入方式 |
|------|------|---------|
| Normal | 执行命令、导航 | `Esc` |
| Insert | 插入文本 | `i`, `a`, `o` 等 |
| Visual | 选择文本 | `v`, `V`, `Ctrl+v` |
| Command-line | 执行命令 | `:` |
| Replace | 替换文本 | `R` |

### 模式切换

```
                  ┌─────────┐
                  │  Normal │
                  └────┬────┘
                       │
          i, a, o, I, A, O, s, S, c, C ───────┐
          R (进入 Replace 模式)                │
          v (字符选择)                         │
          V (行选择)                          ▼
          Ctrl+v (块选择)                  ┌─────────┐
          : (进入 Command-line)             │ Insert  │
          / 或 ? (搜索)                     └─────────┘
          ```

---

## 基本导航

### 在文件中移动

```vim
# 字符导航
h       左移
l       右移
j       下移
k       上移

# 单词导航
w       移到下一个单词开头
W       移到下一个 WORD 开头（以空格分隔）
b       移到当前/上一个单词开头
e       移到当前/下一个单词结尾
E       移到下一个 WORD 结尾

# 行内导航
0       移到行首
^       移到行首（第一个非空字符）
$       移到行尾
gg      移到文件开头
G       移到文件结尾
#行号G  移到指定行

# 屏幕导航
H       移到屏幕顶部
M       移到屏幕中间
L       移到屏幕底部
Ctrl+u  上滚半屏
Ctrl+d  下滚半屏
Ctrl+b  上滚整屏
Ctrl+f  下滚整屏
```

### 搜索导航

```vim
# 基本搜索
/pattern        向前搜索 pattern
?pattern        向后搜索 pattern
n               跳到下一个匹配
N               跳到上一个匹配

# 行内搜索
f{char}         跳到下一个 {char}
F{char}         跳到上一个 {char}
t{char}         跳到下一个 {char} 之前
T{char}         跳到上一个 {char} 之后
;               重复上一次 f/F/t/T
,               反方向重复

# 匹配配对符号
%               在 (), {}, [] 之间跳转
```

---

## 基本编辑

### 文本对象概念

Vim 的编辑基于 **操作符 + 动作** 或 **操作符 + 文本对象** 的组合：

```
操作符 + 动作：d3w（删除3个单词）
操作符 + 文本对象：ci"（修改引号内的内容）
```

### 常用操作符

```vim
d       删除（delete）
y       复制（yank）
c       修改（delete 并进入插入模式）
p       粘贴（paste）
u       撤销（undo）
Ctrl+r  重做（redo）
g~      反转大小写
gU      转为大写
gu      转为小写
>       增加缩进
<       减少缩进
=       自动缩进
```

### 编辑命令示例

```vim
# 删除
dd      删除整行
d$      删除到行尾
dw      删除到下一个单词开头
d0      删除到行首
dgg     删除到文件开头
dG      删除到文件结尾

# 复制
yy      复制整行
y$      复制到行尾
yw      复制到下一个单词结尾

# 修改（删除并进入插入模式）
cc      修改整行
cw      修改到单词结尾
c$      修改到行尾
s       删除当前字符并进入插入模式
S       删除整行并进入插入模式

# 粘贴
p       在光标后粘贴
P       在光标前粘贴

# 重复
.       重复上次操作
```

### 文本对象

```vim
# 内部对象（不包括边界）
iw      当前单词（inner word）
i"      引号内（inner quote）
i'      单引号内
i(      括号内
i[      方括号内
i{      大括号内
ip      当前段落

# 周围对象（包括边界）
aw      整个单词 + 空格
a"      引号及内容
a(      括号及内容
a{      大括号及内容
ap      段落及空行

# 配合操作符使用
ci"     修改引号内的内容
di{     删除大括号内的内容
vi(     选中括号内的内容
yaw     复制整个单词及后面的空格
```

### 插入模式快捷键

```vim
# 进入插入模式
i       在光标前插入
a       在光标后插入
I       在行首插入
A       在行尾插入
o       在下方新建行
O       在上方新建行
s       删除当前字符并插入
S       删除当前行并插入

# 插入模式下移动
左箭头  左移
右箭头  右移
Ctrl+左  按单词左移
Ctrl+右  按单词右移

# 插入模式特殊操作
Ctrl+h  删除前一个字符（退格）
Ctrl+w  删除前一个单词
Ctrl+u  删除到行首
Ctrl+t  增加缩进
Ctrl+d  减少缩进
Esc     退出插入模式
```

---

## 可视模式

### 三种可视模式

```vim
v       字符可视模式 - 按字符选择
V       行可视模式 - 按行选择
Ctrl+v  块可视模式 - 矩形选择
```

### 可视模式操作

```vim
# 在可视模式下
d       删除选中内容
y       复制选中内容
c       修改选中内容
p       用寄存器内容替换选中内容
u       转为小写
U       转为大写
~       反转大小写
>       向右缩进
<       向左缩进

# 可视模式下移动
o       跳到选区另一端
O       在块模式下跳到另一角
gv      重新选择上次选择
```

### 块操作示例

```vim
# 在多行行首添加内容
Ctrl+v        进入块可视模式
jjj           向下选择3行
$             移到行尾
I             进入插入模式
//            输入要添加的内容
Esc           退出，所有行行首都会添加
```

---

## 搜索与替换

### 基本搜索

```vim
# 搜索命令
/正则表达式        向前搜索
?正则表达式        向后搜索

# 搜索选项
:set ignorecase   忽略大小写
:set smartcase    有大写时区分大小写
:set incsearch    增量搜索
:set hlsearch     高亮搜索结果

# 清除搜索高亮
:noh
```

### 替换命令

```vim
# 基本替换
:s/foo/bar/           替换当前行第一个 foo 为 bar
:s/foo/bar/g          替换当前行所有 foo 为 bar
:s/foo/bar/gc         替换前确认
:%s/foo/bar/g         替换文件中所有行
:%s/foo/bar/gc        全局替换前确认

# 范围替换
:5,10s/foo/bar/g      替换第5-10行
:.,$s/foo/bar/g        替换当前行到文件末尾
:%s/foo/bar/g         替换整个文件

# 正则替换
:%s/\(old\)/new/g     使用捕获组
:%s/old/new/g         全局替换
```

### 替换示例

```vim
# 删除行尾空格
:%s/\s+$//

# 交换第一列和第二列
:%s/\([^,]*\),\([^,]*\)/\2,\1/

# 在每行行首添加引号
:%s/^/"/

# 在每行行尾添加引号
:%s/$/"/
```

---

## 多文件操作

### 缓冲区

```vim
# 缓冲区命令
:ls                列出所有缓冲区
:bn                下一个缓冲区
:bp                上一个缓冲区
:b#                切换到上一个缓冲区
:b 3               切换到缓冲区3
:bd                删除缓冲区（关闭文件）
:bd!               强制删除缓冲区
```

### 窗口分割

```vim
# 分割窗口
:sp filename       水平分割（上下）
:vs filename       垂直分割（左右）
:sp               水平分割当前文件
:vs               垂直分割当前文件

# 窗口导航
Ctrl+w h          切换到左侧窗口
Ctrl+w l          切换到右侧窗口
Ctrl+w j          切换到下方窗口
Ctrl+w k          切换到上方窗口
Ctrl+w w          循环切换窗口
Ctrl+w =          所有窗口等宽高
Ctrl+w _          最大化当前窗口高度
Ctrl+w |          最大化当前窗口宽度
```

### 标签页

```vim
:tabnew filename   新建标签页
:tabe filename     在新标签页打开文件
gt                 切换到下一个标签页
gT                 切换到上一个标签页
:tabfirst          切换到第一个标签页
:tablast           切换到最后一个标签页
:tabm 2            将当前标签页移到位置2
:tabs              列出所有标签页
:tabc              关闭当前标签页
```

---

## 宏

宏用于录制和回放一系列操作。

### 基本宏命令

```vim
q{register}        开始录制到寄存器（a-z）
q                 停止录制
@{register}       回放宏
@@                回放上次宏
```

### 宏示例

```vim
# 示例：将多行格式从 var: value 转为 var = value

# 1. 在第一行开始录制
qa

# 2. 执行操作
0               # 移到行首
f:              # 找到冒号
l               # 移到冒号后
x               # 删除冒号
i               # 进入插入模式
 =              # 输入等号
Esc             # 退出插入模式
j               # 移到下一行

# 3. 停止录制
q

# 4. 回放10次
10@a
```

### 宏高级技巧

```vim
# 追加到已有宏
qA              # 追加到寄存器 a（注意大写）

# 查看宏内容
:reg a          # 查看寄存器 a 的内容

# 编辑宏
"ap             # 粘贴宏内容
# 编辑文本
"ayy            # 重新复制
```

---

## 寄存器

寄存器用于存储删除、复制和宏的内容。

### 寄存器类型

```vim
# 无名寄存器（默认）
""               # 最近删除/复制的内容

# 命名寄存器
"a - "z         # 26个命名寄存器
"A - "Z         # 追加到命名寄存器

# 特殊寄存器
"0               # 最近复制的内容
"1 - "9         # 最近9次删除（1是最近）
"-               # 最近一次小删除
".               # 上次插入的文本
":               # 上次命令
"%               # 当前文件名
"#               # 交替文件名
```

### 使用示例

```vim
# 复制到命名寄存器
"ayy             # 复制当前行到寄存器 a
"ap              # 粘贴寄存器 a 的内容

# 清空寄存器
qaq              # 录制空宏

# 交换两行
:dp              # 删除当前行，粘贴
xp               # 交换两个字符
ddp              # 移动当前行
```

---

## 书签与跳转

### 书签

```vim
m{letter}         设置书签（a-z）
`{letter}        跳转到书签
'{letter}        跳转到书签所在行首

# 特殊书签
`.               上次编辑位置
`^               上次插入模式退出位置
`.               上次修改位置
''               上次跳转位置
```

### 跳转列表

```vim
Ctrl+o           后退
Ctrl+i           前进
:jumps           查看跳转列表
:changes         查看修改列表
g;               跳转到旧位置（修改）
g,               跳转到新位置
```

---

## 撤销与重做

```vim
u               撤销
Ctrl+r          重做
U               撤销当前行所有修改

# 撤销树
:undolist       查看撤销列表
:earlier 10m    回到10分钟前
:later 5s       前进5秒
```

---

## 折叠

```vim
# 折叠命令
zf              创建折叠
zd              删除折叠
zD              删除所有折叠
zE              删除窗口中所有折叠

# 打开/关闭折叠
zo              打开当前折叠
zO              打开当前折叠及嵌套
zc              关闭当前折叠
zC              关闭所有嵌套折叠
za              切换打开/关闭
zA              递归切换

# 折叠导航
[z              到折叠开始
]z              到折叠结束
zj              到下一个折叠
zk              到上一个折叠

# 自动折叠
:set foldmethod=manual 手动折叠
:set foldmethod=indent 按缩进折叠
:set foldmethod=syntax 按语法折叠
:set foldmethod=marker 按标记折叠
```

---

## 配置

### .vimrc 基础配置

```vim
" ~/.vimrc 示例

" 基本设置
set number              " 显示行号
set relativenumber      " 显示相对行号
set cursorline          " 高亮当前行
set wrap                " 自动换行
set showcmd             " 显示命令
set wildmenu            " 命令补全

" 缩进设置
set tabstop=4           " Tab 宽度
set shiftwidth=4        " 缩进宽度
set expandtab           " 空格代替 Tab
set smarttab            " 智能 Tab

" 搜索设置
set incsearch           " 增量搜索
set hlsearch            " 高亮搜索
set ignorecase          " 忽略大小写
set smartcase           " 智能大小写

" 编辑设置
set backspace=indent,eol,start  " 退格键行为
set clipboard=unnamed   " 使用系统剪贴板
set autoindent          " 自动缩进
set smartindent         " 智能缩进

" 显示设置
set syntax=on           " 语法高亮
set termguicolors       " True Color
set background=dark     " 深色主题

" 性能设置
set lazyredraw          " 延迟重绘
set ttyfast             " 快速终端
```

### 实用映射

```vim
" 快捷键映射
let mapleader = ","

" 快速保存
nnoremap <leader>w :w<cr>

" 快速退出
nnoremap <leader>q :q<cr>
nnoremap <leader>Q :q!<cr>

" 分屏导航
nnoremap <leader>h :wincmd h<cr>
nnoremap <leader>j :wincmd j<cr>
nnoremap <leader>k :wincmd k<cr>
nnoremap <leader>l :wincmd l<cr>

" Tab 导航
nnoremap <leader>t :tabnext<cr>
nnoremap <leader>T :tabprev<cr>
```

---

## 常用插件

### 插件管理器

```vim
" Vim-Plug 示例
" 在 .vimrc 中添加：
" call plug#begin('~/.vim/plugged')

" Plug 'preservim/nerdtree'       " 文件浏览器
" Plug 'vim-gitgutter'            " Git 集成
" Plug 'tpope/vim-surround'       " 括号/引号操作
" Plug 'junegunn/fzf'             " 模糊搜索
" Plug 'neoclide/coc.nvim'        " 自动补全

" call plug#end()
```

### 常用插件推荐

| 插件 | 用途 |
|------|------|
| NERDTree | 文件浏览器 |
| vim-surround | 快速修改周围字符 |
| fzf.vim | 模糊文件搜索 |
| coc.nvim | LSP 自动补全 |
| vim-commentary | 快速注释 |
| vim-fugitive | Git 集成 |

---

## 技巧汇总

### 常用组合

```vim
# 快速操作
ciw              修改当前单词
caw              修改当前单词（包括空格）
di"              删除引号内内容
yi"              复制引号内内容
vi"              选中引号内内容

# 行操作
dd               删除行
yy               复制行
cc               修改行
>>               增加缩进
<<               减少缩进
==               自动缩进

# 快速更正
xp               交换两个字符
ddp              移动当前行到下一行
dkp              移动当前行到上一行

# 多文件操作
:e filename      打开文件
:bn              下一个文件
:bp              上一个文件
```

### 高效习惯

1. **使用 `.` 命令**：重复上次操作
2. **使用 `f`/`t` 命令**：行内快速跳转
3. **使用文本对象**：`ci(`, `da"`, `yip`
4. **使用宏**：重复复杂操作
5. **使用寄存器**：访问粘贴历史
6. **使用搜索**：快速定位

---

## 参考资料

[^1]: [Vim 官网](https://www.vim.org/)
[^2]: [Vim Galore](https://github.com/mhinz/vim-galore)
[^3]: [Practical Vim](https://pragprog.com/titles/dnvim2/)
