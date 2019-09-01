---
layout: default
---

## Remote Building and installing linux kernel
To remote build the linux kernel and install it to a separate ubuntu machine, first install dependencies.

```
sudo apt-get install build-essential libncurses-dev bison \
                    flex libssl-dev libelf-dev
```

Ubuntu 18 uses the linux kernel 4.15, so we'll grab a base copy of the 4.15.0 kernel:

```
wget https://github.com/torvalds/linux/archive/v4.15.zip
unzip 4.15.zip
cd 4.15.zip
```
then copy the config file used from another operating ubuntu box:
```
cp /boot/config-$(uname -r) .config
```
then prepare the configuration file with:
```
make menuconfig
```
xconfig is also viable, but menuconfig may provide more options. In my opinion,
it is also more stable. If you are trying to develop drivers for fuzzing, usb 
drivers can be found following
```
> Device Drivers > USB support
```
The modules by default are loaded into the linux kernel as one piece, but to
make them separate modules, change the * next to the module to a M, for example:
```
<M>   xHCI HCD (USB 3.0) support
```
This action doesn't seem like much, but if you are trying to fuzz that module,
it helps to have it separate for debugging purposes. For any modifications to
the ethernet devices, namely e1000, changes need to be made in:
```
> Device Drivers > Network device support > Ethernet driver support
```
Now that you have a proper menuconfig built, proceed with a simple make command
```
make -j
```
Now setting the install directory to a location that we can easily copy all the modules from to the location / on the target remote ubuntu box:
```
make INSTALL_MOD_DIR=$(pwd)/../l15inst \
INSTALL_MOD_PATH=$(pwd)/../l15inst modules_install
```


[back](./)
