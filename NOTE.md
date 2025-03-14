# 笔 记

##  jemalloc
checkout 的时候，他用到了 jemalloc 项目，需要git来同步外部引用的库
```shell
    git submodule sync 
```
jemalloc 主要用了内存池来管理内存而不是原有的 malloc管理内存会产生很大量的内存碎片导致内存使用出现异常，还有多线程中 malloc 产生的多线程之间的内存分配的时候产生锁竞争。jemalloc 会提升这部分的性能因为他的每个线程池都会有一个内存池供使用，这样就没有锁竞争了。

## makefile
在 makefile 中，的 include platform.mk 主要处理平台相关的宏定义和 gcc 的基础命令行
其中优先编译 3rd，
### .PHONY
是为了防止文件名与 target 冲突的，如果有一个文件叫做 clean.cpp
```makefile
    all: clean build

    .PHONE: clean build 

    clean:
        rm -rf *.so *.a 
    build:
        gcc ......
```
这样gcc 会认为只去编译 clean.cpp 而不会去找 clean 的 target 执行.

----

## LPeg
这是一个用于做正则表达示实现模式匹配和解析器，比 Lua 中的正则表达式处理，提供更灵活的能力
```lua
local lpeg = require "lpeg"

-- 匹配一个或多个数字
local digit = lpeg.R("09") -- 表示0-9范围内的数字
local number = digit^1      -- ^1 表示至少出现1次

print(number:match("1234")) -- 输出 5 (表示匹配到的位置+1)
print(number:match("abc"))  -- 输出 nil (没有匹配到)
```

```lua
local letter = lpeg.R("az", "AZ")
local identifier = letter * (letter + digit)^0

print(identifier:match("var123")) -- 输出 7
print(identifier:match("123var")) -- 输出 nil

```

---

## 环境变量部分

在 skynet 中环境变量定义的是在 skynet_env.c/h 中定义的，这部分简单就是利用一个 lua_State 放入一些 global 的变量，这些变量可以通过spinlock来限制线程同步获取和设置。本身才去了线程安全。

```C
struct skynet_env {
	struct spinlock lock;
	lua_State *L;
};

static struct skynet_env *E = NULL;

SPIN_LOCK(E) //这个宏的定义在一个专门的spinlock.h文件中，实现的
SPIN_UNLOCK(E)

//具体的 SPIN_LOCK 的实现是采用C11标准库引入的原子操作函数来做的,这里只列出一两个调用的方式
atomic_test_and_set_(&lock->lock);
atomic_exchange_explicit(&lock->lock);
atomic_compare_exchange_weak(&lock->lock);

```


