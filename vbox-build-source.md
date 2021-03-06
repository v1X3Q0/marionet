---
layout: default
---

## Building Virtualbox for ub18 on ub18

```
sudo apt-get install gcc g++ bcc iasl xsltproc uuid-dev \
                zlib1g-dev libidl-dev libsdl1.2-dev libxcursor-dev \
                libasound2-dev libstdc++5 libsdl2-dev \
                libpulse-dev libxml2-dev libxslt1-dev \
                python-dev libqt4-dev qt4-dev-tools libcap-dev \
                libxmu-dev mesa-common-dev libglu1-mesa-dev \
                linux-kernel-headers libcurl4-openssl-dev \
                libpam0g-dev libxrandr-dev libxinerama-dev \
                libqt4-opengl-dev makeself libdevmapper-dev default-jdk \
                texlive-latex-base texlive-latex-extra \
                texlive-latex-recommended texlive-fonts-extra \
                texlive-fonts-recommended libc6-dev-i386 lib32gcc1 \
                gcc-multilib lib32stdc++6 g++-multilib qttools5-dev-tools \
                libssl-dev libvpx-dev libopus-dev \
                libqt5x11extras5-dev libqt5charts5-dev flex bison
```

afterwards link some 32 bit libs for 64 bit accessibility:
```
ln -s libX11.so.6    /usr/lib32/libX11.so 
ln -s libXTrap.so.6  /usr/lib32/libXTrap.so 
ln -s libXt.so.6     /usr/lib32/libXt.so 
ln -s libXtst.so.6   /usr/lib32/libXtst.so
ln -s libXmu.so.6    /usr/lib32/libXmu.so
ln -s libXext.so.6   /usr/lib32/libXext.so
```

modify the file Config.kmk, changing the line:
```
VBOX_JAVAC_OPTS   = -encoding UTF-8 -source 1.5 -target 1.5 -Xlint:unchecked
```

so that both the target and the source are version 1.6. Afterwards configuring
and setting the build environment. Following we build for debug targets:

```
./configure --disable-hardening
source ./env.sh
kmk BUILD_TYPE=debug
```

[back](./)
