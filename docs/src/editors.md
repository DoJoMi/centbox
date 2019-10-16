# Editors

------------------------------------------------------------------------

## Nano

```shell
|:-----------|---------------------:|
|  ctrl v    |   previous page      |
|  ctrl w    |   where is (find)    |
|  ctrl k    |   cut that line      |
|  ctrl x    |   exit the editor    |
|  ALT /     |   end of file        |
|  crtl a    |   beginning of line  |
|  crtl e    |   end of line        |
|  crtl c    |   show line number   |
|  crtl _    |   go to line         |
|:-----------|---------------------:|
```

**For more configuration details** <https://github.com/DoJoMi/dotnano>

## Vi(m)

```shell

|:filemanagement---|---------------------------------------------------:|
|  :e foo          |   load foo                                         |
|  :e path         |   filemanager                                      |            
|  :sav foo        |   save foo                                         |
|  :! bash         |   open terminal inside vim                         |
|  :r !date        |   insert shell output as text                      |
|  :w! sudo tee %  |   save protected file                              |
|:editor-----------|---------------------------------------------------:|
|  d               |   insert new line                                  |
|  u               |   undo                                             |
|  p               |   paste                                            |
|  ctrl+r          |   redo                                             |
|  /foo            |   search foo                                       |
|  /foo/i          |   search foo ignore upper/lower case               |
|  n               |   for the next match                               |
|  :5              |   jump to the 5th line                             |
|  :$              |   jump to the last line                            |
|  .               |   repeat the last command                          |
|  :@              |   repeat the last commandline command              |
|  yy              |   copy line                                        |
|   x              |   delete current character                         |
|  dd              |   delete line                                      |
|  dw              |   cut of word from actual position to right        |
|  5dd             |   delete five lines                                |
|  db              |   cut of word from actual position to left         |
|  d^              |   delete to beginning of line                      |
|  d$              |   delete to end of line                            |
|  :1,.d           |   delete to beginning of file                      |
|  :.,$d           |   delete to end of file                            |
|  gg=G            |   formating code                                   |
|  : 'a,'bd        |   delete from a to b                               |
|  :%s/foo/bar     |   only delete once                                 |
|  :%s/foo/bar/g   |   search & substitute globally                     |
|  :2,5s/foo/bar/  |   from line 2 to 5 substitute foo with bar         |
|:mark-------------|---------------------------------------------------:|
|  v               |   set start of mark area                           |
|  y               |   yank (cp)                                        |
|  VG              |   mark all                                         |
|:navigation-------|---------------------------------------------------:|
|  gg              |   jump to the first position of text               |
|  GG              |   jump to the last position of text                |
|:some other-------|---------------------------------------------------:|
|  :split          |   h split                                          |
|  :syntax on      |   syntax highlight on                              |
|  :set nu!        |   unset linenumber                                 |
|  :set ai         |   auto ident                                       |
|  :set cin        |   auto ident C-Code style                          |
|:-----------------|---------------------------------------------------:|
```

**For more configuration details** <https://github.com/DoJoMi/dotvim>
