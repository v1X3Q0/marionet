---
layout: default
---

The windows PE file format expands across most architecures, including x86,
x86-64 and ARM. When calling CreateProcess on a compiled executable, the loading
sequence opaquely takes a file on the filesystem and creates a process for it.
It loads the process into memory, copying the correct sections to their
requested virtual address in memory, relocating any addresses that need rebasing
and importing any external calls that are referenced. Should a user have a
binary that they would like to create a process for, without creating a pid for
it, they have the ability to perform the above tasks themselves to load one
process into another.

Metasploit has a tool that already perform this technique, reflective injection,
[here](https://www.rapid7.com/db/modules/post/windows/manage/reflective_dll_inject).

This blogpost will be covering a few of the basics of this technique, and expand
on how it can be used to reflectively inject an ARM PE EXE.

The first thing to point out is that there are no evident differences between
PEs across any and all architectures. The NT header, file header, sections and
all necessary offsets are accessible from the Visual Studio ARM SDK. They can
also be referenced by the tool CFF explorer the same way per architecture as
well.

The first noted difference is the architecture field of the file header.
Although ARM binaries will be recognized by CFF explorer as Unknown, a value of
0x1c4 can be seen in this field. Although not necessary to observe during
runtime, it is a useful reference for binaries that can be compiled or ripped
from targets such as windows phones.

![armNtosFH](../assets/images/armNtosFH.jpg)

![amd64NtosFH](../assets/images/amd64NtosFH.jpg)


Moving on to some more major differences between architectures, the relocation
table. I deconstruct the process of relocation in the Windows PE loader, but a
good link for reference is [here](http://research32.blogspot.com/2015/01/base-relocation-table.html?m=1).

The only thing that the post does not go into with depth is the most important
difference, relocation types. Most windows PEs only use the relocation type
IMAGE_REL_BASED_HIGHLOW. ARM adds an additional type that needs relocation of
imports. The reason for this is the loss of variable length instructions. For
calling imports, x86 has the ability to load a relative offset into a register
in one instruction, then dereferencing it to be called, thereby a resolved
import. ARM meanwhile has to load a four byte address, two bytes at a time using
two four byte instructions.

The four byte address loaded is the absolute address should the binary be loaded
at 0x000400000. In the process of reflective injecting, this is fine should
VirtualAlloc with that base and the correct virtual image size specified. Then
you can ignore the relocation table as a whole and use that address for the base
of your PE. But should a single page be mapped in that  range, the whole load
will fail.

So onto the juicy part, relocating these absolute address instructions. This can
be done by decoding the load upper and load lower operations done on these PLT
like routines. These decoded absolute addresses are then subtracted by the
typical base offset and added to the new process base address. They can then be
re-encoded into the load operations and written to memory. That's it!

```c
case IMAGE_REL_BASED_MACHINE_SPECIFIC_7:
    oldAddress = 0;
    dirtyInst = *(size_t*)(relocVirtAddr);
    newAddress = 0;
    oldAddress = oldAddress | ((dirtyInst & 0x0000000f) << 12);
    oldAddress = oldAddress | ((dirtyInst & 0x00000400) << 1);
    oldAddress = oldAddress | ((dirtyInst & 0x00ff0000) >> 16);
    oldAddress = oldAddress | ((dirtyInst & 0x70000000) >> 20);
    dirtyInst = *(size_t*)(relocVirtAddr + 1);
    oldAddress = oldAddress | ((dirtyInst & 0x0000000f) << 28);
    oldAddress = oldAddress | ((dirtyInst & 0x00000400) << 17);
    oldAddress = oldAddress | ((dirtyInst & 0x00ff0000) >> 0);
    oldAddress = oldAddress | ((dirtyInst & 0x70000000) >> 4);
    newAddress = oldAddress + delta;

    dirtyInst = 0x0c00f240;
    dirtyInst = dirtyInst | ((newAddress & 0x0000f000) >> 12);
    dirtyInst = dirtyInst | ((newAddress & 0x000000ff) << 16);
    dirtyInst = dirtyInst | ((newAddress & 0x00000700) << 20);
    dirtyInst = dirtyInst | ((newAddress & 0x00000800) >> 1);
    *relocVirtAddr = dirtyInst;
    relocVirtAddr++;
    dirtyInst = 0x0c00f2c0;
    dirtyInst = dirtyInst | ((newAddress & 0xf0000000) >> 28);
    dirtyInst = dirtyInst | ((newAddress & 0x00ff0000) << 0);
    dirtyInst = dirtyInst | ((newAddress & 0x07000000) << 4);
    dirtyInst = dirtyInst | ((newAddress & 0x08000000) >> 17);
    *relocVirtAddr = dirtyInst;

    break;
```

The final part is reflectuve injecting an EXE vs a DLL. DLLs, or shared
libraries typically have a DllMain as a point of entry, although not required.
Once called, this routine calls any required setup code and routines that the
module requires to be run. The module then remains in memory for any exported
functions in that module to be called.

EXEs by contrast are required to have a point of entry, of which will be called
on start. The entry point routine sets up any handles necessary for the main
routine, such as retrieving the values to be used as argc and argv. This
functionality is implicitly always performed in an executable, even if not
specified for a main routine to have parameters as such for a compiler. The main
routine is then called and returns. Finally, the enrypoint routine will call the
imported function _exit_, which will signal the operating system to delete the
process and clean up its address space.

The underlying problem being that the operating system is now trying to kill the
process that was used to run code, where maybe that was not the desired
functionality. Under the circumstance that the desired executable to be run
must be an EXE, the import for _exit_ should ideally point somewhere else.

For this to be performed, we can iterate imports for our target PE and choose to
write to the address for _exit_'s import a routine that will correctly clean the
stack and registers, while returning to the expected runtime and possibly
killing the thread.

```c
IMAGE_IMPORT_BY_NAME *import_name =
    (IMAGE_IMPORT_BY_NAME*)(ogT->u1.AddressOfData + (size_t)allocBase);
if (strcmp(import_name->Name, "__p___argc") == 0)
    temp_procedure = (FARPROC)pargc;
else if (strcmp(import_name->Name, "__p___argv") == 0)
    temp_procedure = (FARPROC)pargv;
else if (strcmp(import_name->Name, "exit") == 0)
    temp_procedure = (FARPROC)cust_exit;
```

A quick note, the routines ___p___argc_ and ___p___argv_ are used to retrieve
the parameters for the main argc and argv. Overwriting the values where the
current routines argc and argv are located proved difficult, however replacing
the functions that are used to get those parameters seemed to be the simplest
way to go, to return wanted parameters.

A routine that should be inplace of the _exit_ method would have to be able to
find where the stack pointer was, and know how to restore it when resuming
execution. For this, a few globals will be allocated which will hold the stack
pointer before execution, to properly 0restore it. This functionality should look
like:

```c
void cust_exit()
{
	size_t* startStk = (size_t*)*stackPtr;
#ifdef _X86_
	startStk = (size_t*)*stackPtr - 2;
#endif
	size_t retAddress = 0;
	size_t stackTar = 0;
	void* frameTar = 0;
	for (int i = 0; i < 0x100; i++)
		if ((*(startStk - i) > (size_t)*pcPtr) && (*(startStk - i) < ((size_t)*pcPtr) + 0x40))
		{
			retAddress = *(startStk - i);
			stackTar = (size_t)(startStk - i);
			break;
		}
	if (retAddress != 0)
	{
		frameTar = (void*)*stackPtr;
		void(*retAddress_f)(void*, void*) = reinterpret_cast<void(*)(void*, void*)>((void*)retAddress);
		retAddress_f(startStk, frameTar);
	}
}
```

To save a copy of the 

```c
	argcN = &argcM;
	argvN = &argvM;
	stackPtr = (void**)malloc(sizeof(void*));
	pcPtr = (void**)malloc(sizeof(void*));
	char* FSV = (char*)VirtualAlloc(NULL, sizeof(fullSetVar), MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
	RtlCopyMemory(FSV, fullSetVar, sizeof(fullSetVar));
#ifdef _ARM_
	FSV++;
#endif
	void(*fullSetVar_f)(void**, void**, void*) = reinterpret_cast<void(*)(void**, void**, void*)>((void*)FSV);
	fullSetVar_f(stackPtr, pcPtr, (void*)MAIN);
```

This will require us to have a reference for where the stack needs to cleanup
and return. We can achieve this by creating a shellcode wrapper for the target
entry routine that will save the necessary registers to a global variable for
use in our respective _exit_ function.

[back](./)
