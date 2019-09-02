---
layout: default
---

## Reflect loading ELF binaries

ELF shared object files behave very similarly to Windows PE DLLs. One of the
most similar features is the provided API ability to load a library, dlload and
LoadLibrary. A key weakness of dlload is ita lack of the ability to load a
compiled elf executable, to say load or even reflectively inject an executable.
This post will go over implementing a dlload for executable elf files, in
addition to a custom dlsym to aid in retrieving dlload.


[back](./)
