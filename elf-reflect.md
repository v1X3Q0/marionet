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

The first step in this process is identifying the valid offsets required in a
correct ELF header. Assuming a passed in blob is valid, the program header can
be located by examining the elf header using the linux man pages.

```c
ELF header (Ehdr)
    The ELF header is described by the type Elf32_Ehdr or Elf64_Ehdr:

        #define EI_NIDENT 16

        typedef struct {
            unsigned char e_ident[EI_NIDENT];
            uint16_t      e_type;
            uint16_t      e_machine;
            uint32_t      e_version;
            ElfN_Addr     e_entry;
            ElfN_Off      e_phoff;
            ElfN_Off      e_shoff;
            uint32_t      e_flags;
            uint16_t      e_ehsize;
            uint16_t      e_phentsize;
            uint16_t      e_phnum;
            uint16_t      e_shentsize;
            uint16_t      e_shnum;
            uint16_t      e_shstrndx;
        } ElfN_Ehdr;
```

As can the section header. The section header will be used for

[back](./)
