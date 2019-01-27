---
layout: default
---

## How to build gvfsd from source
Building gvfsd is a pain in the dairyaire. When building, a problem ensues from glib headers supported by apt not being up to the version required by gvfsd. Because of this, building may require a temp VM. We begin by making a VMware Workstation ubuntu 18 VM and start fulfilling our dependencies list.
Initial dependencies that need to be installed:

```
sudo apt-get install pkg-config cmake libglib2.0-dev gtk-doc-tools autoconf \
                    libtool libffi-dev libmount-dev libgcrypt20-dev libxml2-dev dbus \
                    libdbus-1-dev libgcr-3-dev libcap-dev libpolkit-gobject-1-dev \
                    libmozjs-52-dev gperf libudev-dev ninja-build libsystemd-dev \
                    libpam0g-dev intltool libsoup2.4-dev libavahi-client-dev \
                    libavahi-glib-dev libgudev-1.0-dev libfuse-dev libudisks2-dev \
                    libimobiledevice-dev libgoa-1.0-dev libsecret-1-dev libbluray-dev \
                    libusb-1.0-0-dev libsmbclient-dev libarchive-dev \
                    libcdio-paranoia-dev libgdata-dev libgphoto2-dev libmtp-dev \
                    libnfs-dev
```

After initial dependencies have been installed, we observe gvfsd's build system. Because it uses meson, we need to run a quick few steps to install meson on our system.

```
cd ~/repos
wget https://github.com/mesonbuild/meson/archive/master.zip
unzip master.zip
rm master.zip
cd meson-master
sudo python3 setup.py install
```

Next we have to satisfy the glib version mismatch.
```
cd ~/repos
wget https://download.gnome.org/sources/glib/2.59/glib-2.59.0.tar.xz
tar -xvf glib-2.59.0.tar.xz
rm glib-2.59.0.tar.xz
cd glib-2.59.0/
./autogen.sh
make
sudo make install
```

After satisfying the glib mismatch, there polkit mismatch that needs to be satisfied. To do this, we first need to build elogind and install its headers.
```
wget https://github.com/elogind/elogind/archive/master.zip
unzip master.zip
rm master.zip
cd elogind-master
./configure
make
sudo make install
```

Then we can build polkit.
```
wget https://www.freedesktop.org/software/polkit/releases/polkit-0.115.tar.gz
tar -xvf polkit-0.115.tar.gz
rm polkit-0.115.tar.gz
cd polkit-0.115
./configure
make
sudo make install
```

That's it, all the dependencies have been fulfilled. We can finally use meson to configure gvfsd. I added the prefix so that the install will be in a local directory for copying. Removing it will allow it to be installed to your system. Ninja is used for building gvfsd.
```
cd ~/repos
wget https://github.com/GNOME/gvfs/archive/master.zip
unzip master.zip
rm master.zip
mkdir -p gvfs-master/build
cd gvfs-master/build
meson --prefix=$(pwd)/install_dir ..
ninja
ninja install
```

[back](./)
