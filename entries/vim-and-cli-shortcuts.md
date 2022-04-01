# Useful shortcut keys for when you're left with only the command line

| Where | Keys | Description |
|-|-|-|
| Shell | Ctrl + A | Move to start of the line |
| Shell | Ctrl + E | Move to end of the line |
| Shell | Ctrl + P / Up arrow | Display previous line in command history. Keep pressing again to go backwards in history upto the first line |
| Shell | Ctrl + N / Down arrow | Display next line in command history. Keep pressing again to go forwards in history upto the current line |
| Shell | Ctrl + R | Brings up command search (reverse-i-search) prompt. Can be used to lookup previously typed commands. <br>Depending upon the shell, can offer more functionality like reverse/forward lookups and traversal on the results. |
| Vim | gg | Go to start of the file |
| Vim | 42 + Shift + g | Go to line number 42 in the file |
| Vim | Shift + g | Go to end of the file |
| Vim | 0 | Go to start of current line |
| Vim | Shift + 4 | Go to end of current line |
| Vim | dd | Delete current line |
| Vim | d$ | Delete everything after the cursor |
| Vim | Shift + d | Delete everything after the cursor |
| Vim | /word | Forward search for the string "word", from the point of the cursor. Use "n" to go to the next item, use "N" to go back to the previous item. |
| Vim | ?word | Reverse search for the string "word", from the point of the cursor. Use "n" to go to the next item, use "N" to go back to the previous item. |
| Vim | : + %s/word/Bird/ | String replacement. "s" indicates substitution. "/" is field separator, could also use other symbols, but "/" is easier to remember. **Global scoped** |
| Vim | : + 10,20s/word/Bird/ | String replacement. "s" indicates substitution. "/" is field separator, could also use other symbols, but "/" is easier to remember. **Scoped to lines between 10 and 20** |
| | | |
> For more Vim shortcuts, use `vimtutor` or open vim and use `:help`  
> Will add more, when I remember/use them.
