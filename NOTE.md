# 笔 记

##  jemalloc
checkout 的时候，他用到了 jemalloc 项目，需要git来同步外部引用的库
```shell
    git submodule sync 
```
jemalloc 主要用了内存池来管理内存而不是原有的 malloc管理内存会产生很大量的内存碎片导致内存使用出现异常，还有多线程中 malloc 产生的多线程之间的内存分配的时候产生锁竞争。jemalloc 会提升这部分的性能因为他的每个线程池都会有一个内存池供使用，这样就没有锁竞争了。
这部分内存分配的逻辑，仅限于在 Linux 环境，其他环境依旧用的是 malloc，另外在看到用 malloc内存分配的时候，用了链表，按照一个


## makefile
在 makefile 中，的 include platform.mk 主要处理平台相关的宏定义和 gcc 的基础命令行
其中优先编译 3rd，

## .PHONY
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

---

## lualib-src/sproto 协议封装
这目录下面的代码是一个独立的项目，他实现的protocol buf, 从他的描述说比 google protocol buffer速度要快，仔细看了代码才知道很多地方抠到了字节。
```C
struct sproto * sproto_create(const void * proto, size_t sz);
void sproto_release(struct sproto *);

int sproto_prototag(const struct sproto *, const char * name);
const char * sproto_protoname(const struct sproto *, int proto);
// SPROTO_REQUEST(0) : request, SPROTO_RESPONSE(1): response
struct sproto_type * sproto_protoquery(const struct sproto *, int proto, int what);
int sproto_protoresponse(const struct sproto *, int proto);

struct sproto_type * sproto_type(const struct sproto *, const char * type_name);

int sproto_pack(const void * src, int srcsz, void * buffer, int bufsz);
int sproto_unpack(const void * src, int srcsz, void * buffer, int bufsz);
```
这是对于每一个 bit 都要处理到位的地方，有很多地方可以学习。这部分逻辑并不复杂，但是应用手法比较苛刻，所以对空间和时间的把握就很好。

---

## Network
 基础的TCP/IP 接口的实现，支持 TCP UDP 的协议。对于 MacosX 和 FreeBSD 系统在编译的时候通过宏定义导向逻辑 KEvent 来实现(socket_server.c, socket_pool.h, socket_kqueue.h)，Linux 下面导向 epoll (socket_epoll.h)实现. 核心函数 send_request

 ```
 send_request(ss, &request, 'O', sizeof(request.u.open) + len); 
 ```
 类似这种调用，其中第三个参数一个字符，代表不同含义：
 ```C
 /*
	The first byte is TYPE
	R Resume socket
	S Pause socket
	B Bind socket
	L Listen socket
	K Close socket
	O Connect to (Open)
	X Exit socket thread
	W Enable write
	D Send package (high)
	P Send package (low)
	A Send UDP package
	C set udp address
	N client dial to UDP host port
	T Set opt
	U Create UDP socket
 */
 ```

 这里注意逻辑上别搞混了，send_request 不是在发网络包，是在向另外一个线程或者进程发送一个包，让这个线程或者进程去做事情，处理这个包的函数是：
 ```C
static int ctrl_cmd(struct socket_server *ss, struct socket_message *result); 
 ```
 根据不同的 Type 做不同的逻辑，有创建连接的，有接受数据的有。UDP 有通过 C 来设置好 UDP 的目标地址，A发送UDP 包，这和 C 创建 TCP 连接是分开的。

```C
switch (type) {
	case 'R':
		return resume_socket(ss,(struct request_resumepause *)buffer, result);
	case 'S':
		return pause_socket(ss,(struct request_resumepause *)buffer, result);
	case 'B':
		return bind_socket(ss,(struct request_bind *)buffer, result);
	case 'L':
		return listen_socket(ss,(struct request_listen *)buffer, result);
	case 'K':
		return close_socket(ss,(struct request_close *)buffer, result);
	case 'O':
		return open_socket(ss, (struct request_open *)buffer, result);
	case 'X':
		result->opaque = 0;
		result->id = 0;
		result->ud = 0;
		result->data = NULL;
		return SOCKET_EXIT;
	case 'W':
		return trigger_write(ss, (struct request_send *)buffer, result);
	case 'D':
	case 'P': {
		int priority = (type == 'D') ? PRIORITY_HIGH : PRIORITY_LOW;
		struct request_send * request = (struct request_send *) buffer;
		int ret = send_socket(ss, request, result, priority, NULL);
		dec_sending_ref(ss, request->id);
		return ret;
	}
	case 'A': {
		struct request_send_udp * rsu = (struct request_send_udp *)buffer;
		return send_socket(ss, &rsu->send, result, PRIORITY_HIGH, rsu->address);
	}
	case 'C':
		return set_udp_address(ss, (struct request_setudp *)buffer, result);
	case 'N':
		return dial_udp_socket(ss, (struct request_dial_udp *)buffer, result);
	case 'T':
		setopt_socket(ss, (struct request_setopt *)buffer);
		return -1;
	case 'U':
		add_udp_socket(ss, (struct request_udp *)buffer);
		return -1;
	default:
		skynet_error(NULL, "socket-server: Unknown ctrl %c.",type);
		return -1;
	};
```
现在清楚了具体做事情为什么不亲自调用函数来做了，因为脚本的进程和处理逻辑的进程是分开的，需要用 send_request 来传递事件。总结一下逻辑：

    脚本->skynet_socket.c->socket_server.c(function) -> pipline -> ctrl_cmd() -> deal detail logic

网络模块的基本逻辑是这样的：
```C
skynet_start.c : static void start(int thread); -> 
skynet_start.c : create_thread(&pid[2], thread_socket, m); ->
skynet_start.c : static void * thread_socket(void *p); ->
skynet_socket.c : int skynet_socket_poll(); ->
socket_server.c : int socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more); ->
socket_server.c : static int ctrl_cmd(struct socket_server *ss, struct socket_message *result); ->
//然后根据收到的不同类型的指令去做不同的事情，指令列表在上面
```
网络模块的基本逻辑，这条线中有很多其他的逻辑，都基本上是附加的，有什么相关的逻辑就可以按照这条线去发展。
