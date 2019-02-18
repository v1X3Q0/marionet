---
layout: default
---

## Debugging efi bios for arm targets
For efi bios on arm, I based my workflow on that as described by [Linaro](https://wiki.linaro.org/LEG/UEFIforQEMU), and the [edk2](https://github.com/tianocore/edk2) efi firmware.

First thing is getting a copy of edk2, and to be safe initiating all submodules in the repo. done by:
```
git clone https://github.com/tianocore/edk2
cd edk2
git submodule update --init --recursive
```

After we have a pulled copy of edk2, the next step would be to build it from source with our arm gcc compilers specified. If you don't have them, they can be retrieved using:

```
sudo apt-get install gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
```

Next to proceed with bulding the Common Libraries for edk2:
```
make -C BaseTools
```

```
source edksetup.sh
export GCC49_AARCH64_PREFIX=aarch64-linux-gnu-
export GCC49_ARM_PREFIX=arm-linux-gnueabihf-
build -a <architecture> -t GCC49 -p ArmVirtPkg/ArmVirtQemu.dsc
```
where <architecture> is either ARM or AARCH64. Next, if you want boot execute:

```
alias runqarm='qemu-system-arm \
    -m 1024 \
    -cpu cortex-a12 \
    -M virt \
    -bios Build/ArmVirtQemu-<arch>/DEBUG_GCC49/FV/QEMU_EFI.fd \
    -serial stdio \
    -drive file=raw.img,format=raw'
$(runqarm)
```

Now that you have started the bios, it luckily will be running from within your shell. This allows us to capture the output and perform something funny on it. For this we will execute the command script and then evaluate the outted file:

```
script edkrunout.txt
runqarm
exit
```

Waiting for a few seconds, we have edk2 to run through everything it has to do in initialization. The core part that matters most is that we can see a funny string that pops up from the output pretty often, a wonderful feature of edk2 that I believe is a small feature that gdb could use, the lack of efi executable recognition! Luckily, edk2 builds DLLs that satisfy symbols for us. We see the string appear in the debugging output specifying the loading address of the module, as well as the dll path. with a simple python script:

```python
import sys
filename = sys.argv[1]
f = open(filename, 'r')
g = f.readlines()
f.close()
f = open(filename + '.asf', 'w')
for i in g:
    first_thing = i.split(" ")[0]
    if first_thing == 'add-symbol-file':
        f.write(i)
f.close()
```

The resulting output file we can take, and throw all the output in it into a local .gdbinit while changing our $HOME .gdbinit to allow the symbol file loading:

```
add-auto-load-safe-path /home/mariomain/repos/edk2/.gdbinit
set auto-load safe-path /
```

Then adding to our qemu boot script -s and -S, we can boot it before the efi files have been loaded, then debug the efi bios!

To debug, make sure that you use gdb-multiarch, then target remote :1234

[back](./)
