---
layout: post
title: "fuzzing 101 Exercise 3"
date: 2023-07-26 19:06 +0700
modified: 2023-07-26 19:17 +0700
description: fuzzing101 Exercise 3的练习
tag:
  - fuzzing
image:
---

# Exercise-3 tcpdump

### 安装

按照教程步骤

![image-20230725084922414](https://img1.imgtp.com/2023/07/27/k5BMOdxA.png)

安装成功

### 语料库创建

![image-20230725191036958](https://img1.imgtp.com/2023/07/27/fvVHZSbA.png)

### 使用ASAN编译

```
cd $HOME/fuzzing_tcpdump/libpcap-1.8.0/
export LLVM_CONFIG="llvm-config-12"
CC=/root/fuzz/AFLplusplus/afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzz_target/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
 
cd $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/
AFL_USE_ASAN=1 CC=/root/fuzz/AFLplusplus/afl-clang-lto ./configure --prefix="$HOME/fuzz_target/fuzzing_tcpdump/install/"
AFL_USE_ASAN=1 make
AFL_USE_ASAN=1 make install
```

这里配置tcpdump的configure时也要加AFL_USE_ASAN=1，因为它的依赖库也加了ASAN

### fuzz

```
	
AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1 afl-fuzz -m none -i $HOME/fuzzing_tcpdump/tcpdump-tcpdump-4.9.2/tests/ -o $HOME/fuzzing_tcpdump/out/ -s 123 -- $HOME/fuzzing_tcpdump/install/sbin/tcpdump -vvvvXX -ee -nn -r @@

```

![image-20230726221143220](https://img1.imgtp.com/2023/07/27/GCpPFBq6.png)

### 动态调试

最后结果没运行出来，但该漏洞为CVE-2017-13028，属于堆溢出

