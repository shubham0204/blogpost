---
title: llama.cpp distributed inference
draft: true
tags:
  - android
  - machine-learning
---
```
git clone https://github.com/ggerganov/llama.cpp --depth=1
cd llama.cpp
```

```bash
export NDK="/home/shubham/android-ndk-r26d"
cd build_android
cmake -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-33 \
    -DLLAMA_STATIC=Off \
    -DLLAMA_RPC=On \
    -DBUILD_SHARED_LIBS=On \
    -DCMAKE_C_FLAGS=-march=armv8.4a+dotprod ..
make
```