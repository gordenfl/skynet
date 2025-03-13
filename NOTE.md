# 笔 记

##  jemalloc
checkout 的时候，他用到了 jemalloc 项目，需要git来同步外部引用的库
```
    git submodule sync 
```
jemalloc 主要用了内存池来管理内存而不是原有的 malloc管理内存会产生很大量的内存碎片导致内存使用出现异常，还有多线程中 malloc 产生的多线程之间的内存分配的时候产生锁竞争。jemalloc 会提升这部分的性能因为他的每个线程池都会有一个内存池供使用，这样就没有锁竞争了。

## makefile
在 makefile 中，的 include platform.mk 主要处理平台相关的宏定义和 gcc 的基础命令行
其中优先编译 3rd，
### .PHONY
是为了防止文件名与 target 冲突的，如果有一个文件叫做 clean.cpp
```
    all: clean build

    .PHONE: clean build 

    clean:
        rm -rf *.so *.a 
    build:
        gcc ......
```
这样gcc 会认为只去编译 clean.cpp 而不会去找 clean 的 target 执行.




