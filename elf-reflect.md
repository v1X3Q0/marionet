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

That program header can be used for segments of the binary. These contain values
such as that for the load offset of the binary, as well as the dynamic segment.

```c
for (i = 0; i < elfHead->e_phnum; i++)
{
    switch (phdr[i].p_type)
    {
        case PT_DYNAMIC:
            dynTab = (struct Elf64Dyn_t*)(phdr[i].p_offset + fileRaw);
            break;
        case PT_LOAD:
            if (loadOff == 0xffffffffffffffff)
                loadOff = phdr[i].p_vaddr;
            else if (loadOff > phdr->p_vaddr)
                loadOff = phdr[i].p_vaddr;
            break;
    }
}
```

The program header also will have implicitly, the size of the resolved image in
memory. While Windows PEs have this value explicitly saved in its header, ELF
files require a loader to deduce this vaalue by taking the last segment in
memory and adding its loaded virtual address to its size, resulting in the
virtual memory required.

```c
int getVirtSize(struct Elf64Phdr_t* phdr, int phdrNum)
{
	int i;
	int virtSz = 0;
	int newVirt = 0;
	for (i = 0; i < phdrNum; i++)
	{
		newVirt = phdr[i].p_vaddr + phdr[i].p_memsz;
		if (virtSz < newVirt)
			virtSz = newVirt;
	}
	return virtSz;
}
```

The retrieved dynTab pointer to the dynamic section will be used to locate the
various tables, such as that for the strings, symbols, and the process's GOT.
Following that step will be loading the dependencies and imports required for
the binary. I could not find a method that did this any cleaner than the one
used here. Re-iterating the dynamic table allows us to load any libraries
that have string table offsets. These entries in the dynamic table have a type
of DT_NEEDED and tend to begin that table.

```c
int i = 0;
struct Elf64Dyn_t* dynEnt;
for (dynEnt = dynTab; dynEnt->dynType != DT_NULL; dynEnt++)
{
    switch (dynEnt->dynType)
    {
        case DT_NEEDED:
            depTab[i] = dlopen(dynEnt->d_val + strTab, RTLD_NOW);
    }
    i++;
}
```

Now that all the required libraries are loaded into memory, we have to resolve
all the function imports. 

[back](./)
