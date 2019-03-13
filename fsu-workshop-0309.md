---
layout: default
---

## Florida State Cyber Security Workshop 03/09/19

jsfunfuzzer
Simple python
Install for python3
```
git clone https://github.com/MozillaSecurity/funfuzz.git
cd funfuzz
sudo python3 setup.py install
```
Since our crash is on firefox 44.0.2, we will try writing a fuzzer for that target, with hopes of reproducing it.
First retrieve the source
```
wget https://ftp.mozilla.org/pub/firefox/releases/44.0.2/source/firefox-44.0.2.source.tar.xz
tar -xvf firefox-44.0.2.source.tar.xz
```
We have to patch the configure file in the source
First, you must attempt a configure in the js/src directory. This will modify the configure file to have a regular expression check moved to a different line than it was at previoudly, from 15834 to 15735. Once this move happens, you need to modify line 15735 to replace instances of [:space:] with [[:space:]]
so that the line looks like:
```
    version=`sed -n 's/^[[:space:]]*#[[:space:]]*define[[:space:]][[:space:]]*U_ICU_VERSION_MAJOR_NUM[[:space:]][[:space:]]*\([0-9][0-9]*\)[[:space:]]*$/\1/p' "$icudir/common/unicode/uvernum.h"`
```
You also may be required to install autoconf-2.13
with
```
sudo apt-get install autoconf2.13
```
After, we begin the build process
```
cd firefox-44.0.2/js/src
autoconf2.13
mkdir build_DBG.OBJ
cd build_DBG.OBJ
../configure --enable-debug --enable-debug-symbols --disable-optimize --prefix=$(pwd)/gk_install
make -j 2
make install
```
With jsfunfuzz installed and the javascript engine built, it’s time to get the fuzzer running.
A proper config needs to be set up for our fuzzer, create a file js.fuzzmanagerconf in the directory of the installed js engine
```
cd gk_install/bin
```
Add to the file:
```
[Main]
platform = x86_64
product = mozilla-central
os = linux
[Metadata]
pathPrefix = /location/of/firefox/download/firefox-44.0.2/js/src/build_DBG.OBJ/gk_install/bin
buildFlags = --disable-optimize --enable-debug
```
We can now execute the fuzzer
```
python3 -m funfuzz.js.loop --random-flags --compare-jit 20 mozilla-central $(pwd)/js
```
Outputted javascript samples are inside of the instance folder
First sample is located in /wtmp1
Has the samples used for fuzzing
Will have dumps in the file core.tgz
Dump of js process to be reproduced

[back](./)
