---
layout: default
---

## Versions:
- llvm-7.0.0
- mesa-18.2.4 
- glew-2.1.0
- glu-9.0.0

## Starting with llvm
**With 6 gigs of ram or more**

```
mkdir build
cmake -DCMAKE_INSTALL_PREFIX=$(pwd)/../../llvm_out_dynall -BUILD_SHARED_LIBS:TRUE ..
```

[back](./)
