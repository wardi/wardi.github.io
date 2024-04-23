" Ian's .vimrc

" Prolog
au BufNewFile,BufRead *.pro			setf prolog

" Whitespace

highlight ExtraWhitespace ctermbg=darkgreen guibg=darkgreen
match ExtraWhitespace /\s\+\%#\@<!$/


" Python hilighting

let python_hilight_all=1

hi pythonStatement	ctermfg=darkred
hi pythonFunction	ctermfg=red
hi pythonConditional	ctermfg=cyan
hi pythonRepeat		ctermfg=yellow
hi pythonString		ctermfg=blue
hi pythonOperator	ctermfg=darkgreen
hi pythonComment	ctermfg=red
hi pythonTodo		ctermfg=yellow
hi pythonNumber		ctermfg=green
hi pythonException	ctermfg=yellow
hi pythonKeyword	ctermfg=darkgrey
hi pythonNormal		ctermfg=white
hi pythonPreCondit	ctermfg=yellow
hi pythonEscape		ctermfg=cyan


" Prolog hilighting

hi prologKeyword	ctermfg=yellow
hi prologOperator	ctermfg=cyan
hi prologNumber		ctermfg=blue
hi prologSpecialCharacter ctermfg=green
hi prologCComment	ctermfg=red
hi prologComment	ctermfg=darkred
hi prologString		ctermfg=blue
hi prologAtom		ctermfg=cyan
hi prologClauseHead	ctermfg=red


" yaml

hi yamlComment	ctermfg=darkcyan

"set cindent
syntax enable
set smartindent
inoremap # X#


" Tip: set indent parameters for python files.
" Version: 0.1
" Date: 13 May 2006
"
" Description: most python scripts use four spaces for indenting, but
"              sometimes you will end up editing a script where tabs
"              are used; in these situations it can be useful to
"              automatically detect whether spaces or tabs were used,
"              and set some parameters (or call some functions) consequently.
"
" Usage: you can put this script in you vimrc and call the PyIndentAutoCfg
"        function with an autocmd associated to python files, or call
"        it manually, or put it in the python.vim syntax script, or... :-)

" Function to set parameters for python scripts that use
" spaces for indention.  This is also the default.  YMMV.
function PySpacesCfg()
  set expandtab " use spaces in place of tabs.
  set tabstop=8 " number of spaces for a tab.
  set softtabstop=4 " number of spaces for a tab in editing operations.
  set shiftwidth=4 " number of spaces for indent (>>, <<, ...)
  syn match pythonNumber "\<0[oO]\=\%(\o\|_\)\+_\@<![Ll]\=\>"
  syn match pythonNumber "\<0[xX]\%(\x\|_\)\+_\@<![Ll]\=\>"
  syn match pythonNumber "\<0[bB][01_]\+_\@<![Ll]\=\>"
  syn match pythonNumber "\<\%([1-9]\%(\d\|_\)*\|0\)_\@<!\>"
  syn match pythonNumber "\<\d\%(\d\|_\)*_\@<![jJ]\>"
  syn match pythonNumber "\<\d\%(\d\|_\)*_\@<![eE][+-]\=\d\%(\d\|_\)*_\@<![jJ]\=\>"
  syn match pythonNumber "\<\d\%(\d\|_\)*_\@<!\.\%([eE][+-]\=\d\%(\d\|_\)*_\@<!\)\=[jJ]\=\%(\W\|$\)\@="
  syn match pythonNumber "\%(^\|\W\)\zs\d*\.\d\%(\d\|_\)*_\@<!\%([eE][+-]\=\d\%(\d\|_\)*_\@<!\)\=[jJ]\=\>"
endfunction

function JSSpacesCfg()
  set expandtab " use spaces in place of tabs.
  set tabstop=8 " number of spaces for a tab.
  set softtabstop=2 " number of spaces for a tab in editing operations.
  set shiftwidth=2 " number of spaces for indent (>>, <<, ...)
endfunction

" Function to set parameters for python scripts that use
" tabs for indention.  YMMV.
function PyTabsCfg()
  set noexpandtab
  set tabstop=8
  set softtabstop=8
  set shiftwidth=8
endfunction

" This function returns 1 if the file looks like a python script
" that uses tabs for indenting, or 0 otherwise.
function PyIsTabIndent()
  let lnum = 1
  let max_lines = 600 " max number of lines to check.
  let got_tabs = 0
  let got_cols = 0 " 1 if the previous lines ended with columns.
  while lnum <= max_lines
    let line = getline(lnum)
    let lnum = lnum + 1
    if got_cols == 1
      if line =~ "^\t\t" " at least two tabs, to prevent false-positives.
        let got_tabs = 1
        break
      endif
    endif
    if line =~ ":\s*$"
      let got_cols = 1
    else
      let got_cols = 0
    endif
  endwhile

  return got_tabs
endfunction

" Check the file, and run the relative function.
function PyIndentAutoCfg()
  if PyIsTabIndent() == 1
    call PyTabsCfg()
  else
    call PySpacesCfg()
  endif

endfunction

" Call the PyIndentAutoCfg function.  Uncomment this line if you've copied
" this script in the python.vim syntax file or something like that.
" call PyIndentAutoCfg() 
"
autocmd BufNewFile,BufRead *.py call PySpacesCfg()
autocmd BufNewFile,BufRead *.rst call PySpacesCfg()
autocmd BufNewFile,BufRead *.scss call PySpacesCfg()
autocmd BufNewFile,BufRead *.html call JSSpacesCfg()
autocmd BufNewFile,BufRead *.html set filetype=html.jinja
"autocmd BufNewFile,BufRead *.html call jinja#AdjustFiletype()
autocmd BufNewFile,BufRead *.css call JSSpacesCfg()
autocmd BufNewFile,BufRead *.yaml call JSSpacesCfg()
autocmd BufNewFile,BufRead *.yml call JSSpacesCfg()
autocmd BufNewFile,BufRead *.js call JSSpacesCfg()
autocmd BufNewFile,BufRead *.json call JSSpacesCfg()
autocmd BufNewFile,BufRead *.md call JSSpacesCfg()
autocmd BufNewFile,BufRead *.less call JSSpacesCfg()


