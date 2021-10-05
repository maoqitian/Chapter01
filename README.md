>本项目clone于[极客时间Android开发高手课例子](https://github.com/AndroidAdvanceWithGeektime/Chapter01)，该项目集成了[Breakpad](https://github.com/google/breakpad) 来获取发生 native crash 时候的系统信息和线程堆栈信息

## 对原项目更新地方

- 重新放入最新版本的 [Breakpad](https://github.com/google/breakpad) 项目源码
> 原项目不做最新源码引入更改无法在真机上获取dump文件

- 重新编译生成 minidump_stackwalker文件

- 编译环境

依赖库、IDE | version
---|---
NDK | 21.0.6113669
Gradle | 7.0.2
Androidstudio | Arctic Fox(2020.3.1)

## 分析过程

### 运行项目
- 运行项目，点击界面 crash 项目崩溃，会在 /data/data/com.dodola.breakpad/cache/crashDump目录下生成对应 dump 文件，本项目根目录也上传了整个 dump 文件（a9521cfd-1aa8-4192-02da2290-600b74f6.dmp）

![dump文件生成](https://github.com/maoqitian/MaoMdPhoto/raw/master/AndroidNativeCrash/dump%E6%96%87%E4%BB%B6%E7%94%9F%E6%88%90.png)

### 重新编译 minidump_stackwalker
- 我们需要使用 minidump_stackwalker 来分析上一步生成的 
- 原项目 minidump_stackwalker 在 mac 下使用会直接报错，所以需要重新clone [Breakpad](https://github.com/google/breakpad)项目
- 在该项目根目录下自行编译

```
./configure && make
```
- 本项目 tools/mac/minidump_stackwalker 文件已经重新编译好的，适用于 mac 版本，其他其他朋友可自行编译
- 编译完成找到 minidump_stackwalker
![重新编译生成minidump_stackwalker](https://github.com/maoqitian/MaoMdPhoto/raw/master/AndroidNativeCrash/%E9%87%8D%E6%96%B0%E7%BC%96%E8%AF%91%E7%94%9F%E6%88%90%E7%9A%84minidump_stackwalker.png)

### 处理 dump 文件，获取crash日志

- 使用上一步生成的 minidump_stackwalker 工具

```
tools/mac/minidump_stackwalk a9521cfd-1aa8-4192-02da2290-600b74f6.dmp >crash.txt
```
- 在当前目录生成 crash.txt 文件

```
Operating system: Android
                  0.0.0 Linux 4.19.113-perf-g5b8235a #1 SMP PREEMPT Tue Aug 31 02:36:57 CST 2021 aarch64
CPU: arm64
     8 CPUs

GPU: UNKNOWN

Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0
Process uptime: not available

Thread 0 (crashed)
 0  libcrash-lib.so + 0x650
     x0 = 0xb400007d8dee43c0    x1 = 0x0000007fde2ce504
     x2 = 0xb400007d00430000    x3 = 0x0000007fde2cd258
     x4 = 0xb400007cf16a2a10    x5 = 0xe9854b1ddffe04d5
     x6 = 0x0000007fde2cd1e0    x7 = 0x0000007cfd5fa1fc
     x8 = 0x0000000000000000    x9 = 0x0000000000000001
    x10 = 0x0000000000430000   x11 = 0x0000007d80000000
    x12 = 0x000000004119a1b0   x13 = 0x1dcfafe57ab59443
    x14 = 0x0000000000000006   x15 = 0xffffffffffffffff
    x16 = 0x0000007cf32cbfe8   x17 = 0x0000007cf32ca63c
   ........
```

### 分析找到 crash 发生的方法

- 上一步可以得到对应的 CPU 架构是 arm64
- 发生crash原因 SIGSEGV /SEGV_MAPERR，常用型号类型可看这篇文章（[Android 平台 Native 代码的崩溃捕获机制及实现
](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w)）
- 主要关键为以下信息

```
Thread 0 (crashed)
 0  libcrash-lib.so + 0x650
```
- 代表发生crash的动态链接库，还有对应的符号，符号解析，可以使用 ndk 中提供的addr2line来根据地址进行一个符号反解，前面我们知道是arm64 CPU，所以使用NDK的aarch64-linux-android-4.9目录下的addr2line，注意需要添加上 
-f -C -e 命令
```
/xxxxx/android-ndk-r21b/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-addr2line -f -C -e /xxxxxx/sample/build/intermediates/cmake/debug/obj/arm64-v8a/libcrash-lib.so 0x650
```
- 如下最后打印的结果，找出crash方法
![找出crash方法](https://github.com/maoqitian/MaoMdPhoto/raw/master/AndroidNativeCrash/%E6%89%BE%E5%87%BAcrash%E6%96%B9%E6%B3%95.png)
