---
layout: post
title: "fuzzing 101 Exercise 1"
date: 2023-07-08 17:06 +0700
modified: 2023-07-08 19:17 +0700
description: fuzzing101 Exercise 1的练习
tag:
  - fuzzing
image:
---

# Exercise 1-Xpdf

#### 1.Xpdf安装

让我们首先获取模糊目标。为要模糊的项目创建一个新目录：

```
cd $HOME
mkdir fuzzing_xpdf && cd fuzzing_xpdf/
```



要使您的环境完全准备就绪，您可能需要安装一些其他工具（即 make 和 gcc）

```
sudo apt install build-essential
```



下载 Xpdf 3.02：

```
wget https://dl.xpdfreader.com/old/xpdf-3.02.tar.gz
tar -xvzf xpdf-3.02.tar.gz
```



构建 Xpdf：

```
cd xpdf-3.02
sudo apt update && sudo apt install -y build-essential gcc
./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```



是时候测试构建了。首先，您需要下载一些PDF示例：

```
cd $HOME/fuzzing_xpdf
mkdir pdf_examples && cd pdf_examples
wget https://github.com/mozilla/pdf.js-sample-files/raw/master/helloworld.pdf
wget http://www.africau.edu/images/default/sample.pdf
wget https://www.melbpc.org.au/wp-content/uploads/2017/10/small-example-pdf-file.pdf
```



现在，我们可以通过以下方式测试 pdfinfo 二进制文件：

```
$HOME/fuzzing_xpdf/install/bin/pdfinfo -box -meta $HOME/fuzzing_xpdf/pdf_examples/helloworld.pdf
```

安装成功

https://img1.imgtp.com/2023/07/08/gsT5zyP6.png

#### 2.安装afl++

#### 3.fuzz(1)

AFL 是一个覆盖引导的fuzzer，这意味着它为每个变异的输入收集覆盖信息，以便发现新的执行路径和潜在的错误。当源代码可用时，AFL 可以使用检测，在每个基本块（函数、循环等）的开头插入函数调用。

要为我们的目标应用程序启用插桩，我们需要使用 AFL 的编译器编译代码。

```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
```

现在我们将使用 **afl-clang-fast** 编译器构建 xpdf:

```
export LLVM_CONFIG="llvm-config-11"
CC=$HOME/AFLplusplus/afl-clang-fast CXX=$HOME/AFLplusplus/afl-clang-fast++ ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

现在，您可以使用以下命令运行 Fuzzer:

```
afl-fuzz -i $HOME/fuzzing_xpdf/pdf_examples/ -o $HOME/fuzzing_xpdf/out/ -s 123 -- $HOME/fuzzing_xpdf/install/bin/pdftotext @@ $HOME/fuzzing_xpdf/output
```

每个选项的简要说明:

- -i 表示我们必须放置输入用例的目录(a.k.a 文件示例)
- -o 表示 AFL + + 将存储变异文件的目录
- -s 表示要使用的静态随机种子

@@是占位符目标的命令行，AFL 将用每个输入文件名替换它

所以,基本上fuzzer为每个不同的输入文件将运行该命令`$HOME/fuzzing_xpdf/install/bin/pdftotext <input-file-name> $HOME/fuzzing_xpdf/output`

https://img1.imgtp.com/2023/07/08/kng4A0kY.png

成功

https://img1.imgtp.com/2023/07/08/6oBdCt5W.png

> 附：界面信息介绍
>
> **process timing**：执行时间信息
>
> - run time：运行总时间
> - last new find：距离最近一次发现新路径的时间
> - last saved crash：距离最近一次保存程序崩溃的时间
> - last saved hang：距离最近一次保存挂起的时间
>
> **overall results：**
>
> - cycles done：运行的总周期数
> - corpus count：语料库计数
> - saved crashes：保存的程序崩溃个数
> - saved hang：保存的挂起个数
>
> **cycle progress**：
>
> - now processing：当前的测试用例ID（所在输入队列的位置）
> - runs timed out：超时数量
>
> **map coverage：**覆盖率
>
> - map density：目前已经命中多少分支元组，与位图可以容纳多少的比例
> - count coverage：位图中每个被命中的字节平均改变的位数
>
> **stage progress：**
>
> - now trying: 指明当前所用的变异输入的方法
> - stage execs: 当前阶段的进度指示
> - total execs: 全局的进度指示
> - exec speed: 执行速度
>
> **findings in depth：**种子变异产生的信息
>
> - favored items: 基于最小化算法产生新的更好的路径
> - new edges on: 基于更好路径产生的新边
> - total crashes: 基于更好路径产生的崩溃
> - total tmouts: 基于更好路径产生的超时 包括所有超时的超时
>
> **fuzzing strategy yields：** 进一步展示了AFL所做的工作，在更有效路径上得到的结果比例，对应上面的now trying
>
> - bit flips: 比特位翻转，例如：
>
>   - bitflip 1/1，每次翻转1个bit，按照每1个bit的步长从头开始
>   - bitflip 2/1，每次翻转相邻的2个bit，按照每1个bit的步长从头开始
>   - …..
>
> - byte flips: 字节翻转
>
> - arithmetics: 算术运算，例如：
>
>   - arith 16/8，每次对16个bit进行加减运算，按照每8个bit的步长从头开始，即对文件的每个word进行整数加减变异
>
> - know ints: 用于替换的基本都是可能会造成溢出的数，例：
>
>   - interest 16/8，每次对16个bit进替换，按照每8个bit的步长从头开始，即对文件的每个word进行替换
>
> - dictionary: 有以下子阶段：
>
>   - user extras (over)，从头开始，将用户提供的tokens依次替换到原文件中
>
>   - user extras (insert)，从头开始，将用户提供的tokens依次插入到原文件中
>
>   - auto extras (over)，从头开始，将自动检测的tokens依次替换到原文件中
>
>     其中，用户提供的tokens，是在词典文件中设置并通过-x选项指定的，如果没有则跳过相应的子阶段。
>
> - havoc：顾名思义，是充满了各种随机生成的变异，是对原文件的“大破坏”。具体来说，havoc包含了对原文件的多轮变异，每一轮都是将多种方式组合（stacked）而成
>
>   splice：在任意选择的中点将队列中的两个随机输入拼接在一起.
>
> - py/custom/req：
>
> - trim：修建测试用例使其更短，但保证裁剪后仍能达到相同的执行路径
>
> - eff
>
> **item geometry：**
>
> - levels: 表示测试等级
> - pending: 表示还没有经过fuzzing的输入数量
> - pend fav: 表明fuzzer感兴趣的输入数量
> - own finds: 表示在fuzzing过程中新找到的，或者是并行测试从另一个实例导入的数量
> - imported: n/a表明不可用，即没有导入
> - stability: 表明相同输入是否产生了相同的行为，一般结果都是100%

你可以看到 `saved crashes` 值为红色，表示发现的crash的数量。您可以在`$HOME/fuzzing _ xpdf/out/`目录中找到这些崩溃文件。一旦找到第一个crash，你就可以关掉警报器了，这就是我们要解决的。在出现crash之前，根据机器性能的不同，可能需要一到两个小时。

在这个阶段，你已经学到了:

- 如何使用使用检测的 afl 编译器编译目标
- 如何启动 afl + +
- 如何检测您的目标的独特崩溃

接下来呢？我们没有关于这个 bug 的任何信息，只是程序崩溃了… 现在是进行调试和分类的时候了！

#### 4.重现崩溃

https://img1.imgtp.com/2023/07/08/yHWCX5NZ.png

```
$HOME/fuzzing_xpdf/install/bin/pdftotext 'id:000000,sig:11,src:000902,time:370856,execs:210977,op:havoc,rep:16' $HOME/fuzzing_xpdf/output
```

https://img1.imgtp.com/2023/07/08/J5BRHDrW.png

https://img1.imgtp.com/2023/07/08/sIjJHT0M.png

#### 5.调试崩溃

在使用gdb调试前，使用`-g -O0`选项重建`Xpdf`

```
rm -r $HOME/fuzzing_xpdf/install
cd $HOME/fuzzing_xpdf/xpdf-3.02/
make clean
CFLAGS="-g -O0" CXXFLAGS="-g -O0" ./configure --prefix="$HOME/fuzzing_xpdf/install/"
make
make install
```

开始使用gdb调试：

```
gdb --args $HOME/fuzzing_xpdf/install/bin/pdftotext $HOME/fuzzing_xpdf/out/default/crashes/22 $HOME/fuzzing_xpdf/output
```

https://img1.imgtp.com/2023/07/08/SbgBlO4d.png

https://img1.imgtp.com/2023/07/08/yjYZjpK3.png

用bt查看调用栈

https://img1.imgtp.com/2023/07/08/fnf75QqT.png

可以观察到多次在最后调用了`Parser::getObj`，似乎进入了一个无限递归。

#### 6.修复

下载修复了该CVE的`xpdf 4.02`进行对比，

修复方式：多了一个变量记录循环次数，超过一定次数就结束进程。

#### 7.遇到的问题

##### 1.在执行afl-fuzz...后

```
[-] Hmm, your system is configured to send core dump notifications to an
    external utility. This will cause issues: there will be an extended delay
    between stumbling upon a crash and having this information relayed to the
    fuzzer via the standard waitpid() API.
    If you're just testing, set 'AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1'.

    To avoid having crashes misinterpreted as timeouts, please log in as root
    and temporarily modify /proc/sys/kernel/core_pattern, like so:

    echo core >/proc/sys/kernel/core_pattern

[-] PROGRAM ABORT : Pipe at the beginning of 'core_pattern'
         Location : check_crash_handling(), src/afl-fuzz-init.c:2256

```

解决方案：添加AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1

https://img1.imgtp.com/2023/07/08/3w4Bw34r.png