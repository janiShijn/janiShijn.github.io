---
layout: post
title: "fuzzing 101 Exercise 2"
date: 2023-07-24 19:06 +0700
modified: 2023-07-24 19:17 +0700
description: fuzzing101 Exercise 2的练习
tag:
  - fuzzing
image:
---

# Exercise-2 libexif

#### 1.安装libexif

[模糊测试101/练习2在主·安东尼奥-莫拉莱斯/模糊101 ·GitHub](https://github.com/antonio-morales/Fuzzing101/tree/main/Exercise 2)

##### 下载并构建您的目标

让我们首先获取模糊目标。为我们要模糊的项目创建一个新目录：

```
cd $HOME
mkdir fuzzing_libexif && cd fuzzing_libexif/
```

##### 下载并解压缩 libexif-0.6.14：

```
wget https://github.com/libexif/libexif/archive/refs/tags/libexif-0_6_14-release.tar.gz
tar -xzvf libexif-0_6_14-release.tar.gz
```

##### 构建并安装 libexif：

```
cd libexif-libexif-0_6_14-release/
sudo apt-get install autopoint libtool gettext libpopt-dev
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
```

##### 选择接口应用程序

由于 libexif 是一个库，我们需要另一个利用这个库并且将被模糊处理的应用程序。对于此任务，我们将使用 **exif 命令行**。键入以下内容以下载和解压缩 exif 命令行 0.6.15：

```
cd $HOME/fuzzing_libexif
wget https://github.com/libexif/exif/archive/refs/tags/exif-0_6_15-release.tar.gz
tar -xzvf exif-0_6_15-release.tar.gz
```



##### 现在，我们可以构建并安装 exif 命令行实用程序：

```
cd exif-exif-0_6_15-release/
autoreconf -fvi
./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```



##### 要测试一切是否正常工作，只需键入：

```
$HOME/fuzzing_libexif/install/bin/exif
```

![image](https://img1.imgtp.com/2023/07/25/lvzzg0qL.png)

#### 2.示例安装

```
cd $HOME/fuzzing_libexif
wget https://github.com/ianare/exif-samples/archive/refs/heads/master.zip
unzip master.zip
```

例如，我们可以执行以下操作：

```
$HOME/fuzzing_libexif/install/bin/exif $HOME/fuzzing_libexif/exif-samples-master/jpg/Canon_40D_photoshop_import.jpg
```

输出：

![image](https://img1.imgtp.com/2023/07/25/bdNgTm2X.png)

#### 3.用afl-clang-lto作为编译器来构建libexif

现在我们将使用 **afl-clang-lto** 作为编译器来构建 libexif。

```
rm -r $HOME/fuzzing_libexif/install
cd $HOME/fuzzing_libexif/libexif-libexif-0_6_14-release/
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/"
make
make install
```

```
cd $HOME/fuzzing_libexif/exif-exif-0_6_15-release
make clean
export LLVM_CONFIG="llvm-config-11"
CC=afl-clang-lto ./configure --enable-shared=no --prefix="$HOME/fuzzing_libexif/install/" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
make install
```

如您所见，我使用了afl-clang-lto而不是**afl-clang-fast**。一般来说，*afl-clang-lto* 是最好的选择，因为它是一种无碰撞的仪器，而且比 *afl-clang-fast* 更快。

如果您不确定何时使用 afl-clang-lto 或 *afl-clang-fast*，您可以查看从 [AFLplusplus](https://github.com/AFLplusplus/AFLplusplus#1-instrumenting-that-target) 中提取的下图： 检测该目标

```
+--------------------------------+
| clang/clang++ 11+ is available | --> use LTO mode (afl-clang-lto/afl-clang-lto++)
+--------------------------------+     see [instrumentation/README.lto.md](instrumentation/README.lto.md)
    |
    | if not, or if the target fails with LTO afl-clang-lto/++
    |
    v
+---------------------------------+
| clang/clang++ 6.0+ is available | --> use LLVM mode (afl-clang-fast/afl-clang-fast++)
+---------------------------------+     see [instrumentation/README.llvm.md](instrumentation/README.llvm.md)
    |
    | if not, or if the target fails with LLVM afl-clang-fast/++
    |
    v
 +--------------------------------+
 | gcc 5+ is available            | -> use GCC_PLUGIN mode (afl-gcc-fast/afl-g++-fast)
 +--------------------------------+    see [instrumentation/README.gcc_plugin.md](instrumentation/README.gcc_plugin.md) and
                                       [instrumentation/README.instrument_list.md](instrumentation/README.instrument_list.md)
    |
    | if not, or if you do not have a gcc with plugin support
    |
    v
   use GCC mode (afl-gcc/afl-g++) (or afl-clang/afl-clang++ for clang)
```

#### 4.模糊测试

现在，您可以使用以下命令运行模糊程序：

```
afl-fuzz -i $HOME/fuzzing_libexif/exif-samples-master/jpg/ -o $HOME/fuzzing_libexif/out/ -s 123 -- $HOME/fuzzing_libexif/install/bin/exif @@
```

几分钟后，出现多次崩溃：

```
AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1 afl-fuzz -i $HOME/fuzzing_libexif/exif-samples-master/jpg/ -o $HOME/fuzzing_libexif/out/ -s 123 -- $HOME/fuzzing_libexif/install/bin/exif @@
```

![image](https://img1.imgtp.com/2023/07/25/uQSxZ1ay.png)

![image](https://img1.imgtp.com/2023/07/25/vVQ87a1x.png)

#### 5.动态调试

先重新编译带调试信息的`libexif`和`exif`，这样就可以源码调试了：

```bash
# libexif
rm -r $HOME/fuzzing_libexif/install
cd $HOME/fuzzing_libexif/libexif-libexif-0_6_14-release/
make clean
CFLAGS="-g -O0" CXXFLAGS="-g -O0"
make

# exif
cd $HOME/fuzzing_libexif/exif-exif-0_6_15-release
make clean
CFLAGS="-g -O0" CXXFLAGS="-g -O0" PKG_CONFIG_PATH=$HOME/fuzzing_libexif/install/lib/pkgconfig
make
```

通过对报错输入的调试，主要有以下两条执行流：

```bash
pwndbg> bt
#0  0x000000000022c1bf in exif_get_sshort (buf=0x100451e85 <error: Cannot access memory at address 0x100451e85>, order=EXIF_BYTE_ORDER_MOTOROLA) at exif-utils.c:92
#1  exif_get_short (buf=0x100451e85 <error: Cannot access memory at address 0x100451e85>, order=EXIF_BYTE_ORDER_MOTOROLA) at exif-utils.c:104
#2  exif_data_load_data (data=0x452680, d_orig=<optimized out>, ds_orig=<optimized out>) at exif-data.c:819
#3  0x00000000002214d6 in exif_loader_get_data (loader=<optimized out>) at /home/pursue/桌面/Fuzzing101-main/Exercise 2/fuzzing_libexif/libexif-0.6.14/libexif/exif-loader.c:387
#4  main (argc=<optimized out>, argv=<optimized out>) at main.c:438
#5  0x00007ffff7cb7d90 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#6  0x00007ffff7cb7e40 in __libc_start_main () from /lib/x86_64-linux-gnu/libc.so.6
#7  0x000000000021a925 in _start ()

pwndbg> bt
#0  0x00007ffff7e2eed0 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x000000000022efe5 in exif_data_load_data_thumbnail (data=0x452610, d=0x451e16 "MM", ds=2026, offset=702, size=4294967168) at exif-data.c:292
#2  exif_data_load_data_content (data=<optimized out>, ifd=<optimized out>, d=<optimized out>, ds=<optimized out>, offset=674, recursion_depth=<optimized out>) at exif-data.c:381
#3  0x000000000022c322 in exif_data_load_data (data=0x452610, d_orig=<optimized out>, ds_orig=<optimized out>) at exif-data.c:835
#4  0x00000000002214d6 in exif_loader_get_data (loader=<optimized out>) at /home/pursue/桌面/Fuzzing101-main/Exercise 2/fuzzing_libexif/libexif-0.6.14/libexif/exif-loader.c:387
#5  main (argc=<optimized out>, argv=<optimized out>) at main.c:438
#6  0x00007ffff7cb7d90 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#7  0x00007ffff7cb7e40 in __libc_start_main () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x000000000021a925 in _start ()
```

- 第一条执行流：`main -> exif_loader_get_data -> exif_data_load_data -> exif_get_short -> exif_get_sshort`

  可以在回溯中看到buf越权访问：

  ```bash
  0x000000000022c1bf in exif_get_sshort (buf=0x100451e85 <error: Cannot access memory at address 0x100451e85>, order=EXIF_BYTE_ORDER_MOTOROLA) at exif-utils.c:92		# Cannot access memory at address 0x100451e85
  ```

  往回找一下，在`exif-data.c`的819行找到了buf的原因：

  ```c
      /* IFD 1 offset */
  // ExifLong offset; unsigned int ds;
      if (offset + 6 + 2 > ds) {		// 检查
          return;
      }
      n = exif_get_short (d + 6 + offset, data->priv->order);
  ```

  由于这里因为gdb优化代码编译的原因，所以我们不能从回溯中找到参数的信息。这里通过gdb的动调，找到了`offset`和`ds`的值。

  ```bash
  # pwndbg
  ──────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────
  *RBX  0x7c3
  *RBP  0xffffffff
  ───────────────────────────────────[ DISASM / x86-64 / set emulate on ]────────────────────────────────────
   ► 0x22c117 <exif_data_load_data+1239>    lea    eax, [rbp + 8]                <__afl_area_initial>
     0x22c11a <exif_data_load_data+1242>    cmp    eax, ebx
     0x22c11c <exif_data_load_data+1244>    jbe    exif_data_load_data+1306   
  ```

  可以看到`offset`的值为`0xffffffff`，也就是`-1`，加上8那就是`7`；而`ds`的值是`0x7c3`，显然是可以通过检查的。总结那就是当`offset`的值在`-8 ~ -1`之间是能够绕过检查触发程序崩溃的。所以修改如下：

  ```c
  if (offset > 0xfffffff0 | offset + 6 + 2 > ds) {
      return;
  }
  ```

- 第二条执行流：`main -> exif_loader_get_data -> exif_data_load_data -> exif_data_load_data_content -> exif_data_load_data_thumbnail`

  可以在回溯中清晰的看到`memcpy()`函数复制的size过大导致崩溃：

  ```bash
  # pwndbg
     288 	data->size = size;
     289 	data->data = exif_data_alloc (data, data->size);
     290 	if (!data->data) 
     291 		return;
   ► 292 	memcpy (data->data, d + offset, data->size);
     293 }
  
  # bt
  0x000000000022efe5 in exif_data_load_data_thumbnail (data=0x452610, d=0x451e16 "MM", ds=2026, offset=702, size=4294967168) at exif-data.c:292	# size=4294967168
  ```

  以下是`exif_data_load_data_thumbnail`函数的源码：

  ```c
  static void
  exif_data_load_data_thumbnail (ExifData *data, const unsigned char *d,
                     unsigned int ds, ExifLong offset, ExifLong size)
  {
      // typedef uint32_t	ExifLong;          /* 4 bytes */
      if (ds < offset + size) {
          // 在这里，ds=2026, offset=702, size=0xFFFFFF80=-128，显然是可以通过检查的
          exif_log (data->priv->log, EXIF_LOG_CODE_DEBUG, "ExifData",
                "Bogus thumbnail offset and size: %i < %i + %i.",
                (int) ds, (int) offset, (int) size);
          return;
      }
      if (data->data) 
          exif_mem_free (data->priv->mem, data->data);
      /*
      typedef struct _ExifData        ExifData; 
      struct _ExifData
      {
          ExifContent *ifd[EXIF_IFD_COUNT];
          unsigned char *data;
          unsigned int size;		// 问题就在此处
          ExifDataPrivate *priv;
      };
      */
      data->size = size;		// 强制转换的时候出现了问题
      // size=0xFFFFFF80=4294967168
      data->data = exif_data_alloc (data, data->size);
      if (!data->data) 
          return;
      memcpy (data->data, d + offset, data->size);
  }
  ```

  所以只需要更新一下这个检查就行：

  ```c
  if ((unsigned long int) ds < (unsigned long int) offset + (unsigned long int) size){
      ...
  }
  ```

这两个洞都和**整数溢出**有关。

#### 6.漏洞

 libexif 0.6.18 中的 libexif/exif-entry.c 中的 exif_entry_fix 函数（又名标签修复例程）中基于堆的缓冲区溢出允许远程攻击者导致拒绝服务或可能通过无效的 EXIF 图像执行任意代码。注意：其中一些详细信息是从第三方信息获得的。



 libexif中的 exif-data.c 中的 exif_data_load_data 函数允许远程攻击者造成拒绝服务（越界读取），或者可能通过图像中精心设计的 EXIF 标签从进程内存中获取敏感信息。
