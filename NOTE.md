# Skynet code review note

## 启动逻辑
- skynet_global_initial pthread 创建线程，设置主线程相关变量
- skynet_env_initial 在主线程创建 LuaState 然后创建一个 Lua 虚拟机对象（并没有 load 任何人 config 数据）
- sigign 创建 signal 的反馈函数，主要包含的信号量包括：SIG_IGN， SIGPIPE
- 如果LUA_CACHELIB 宏定义了启用 Lua 代码缓存功能（目的是：提高 lua 代码的加载效率）
- 以代码中的字符串初始化配置文件Lua 虚拟机中，关闭这个 Lua 虚拟机（有点绕是吧，但是有点想法，是为了不被干扰，保持配置文件模块中的虚拟机干净整齐）
- 调用skynet_star(struct skynet_config * config) 开始启动
- 先绑定 SIGNEL 的处理函数， 然后初始化，如果 config 有定义 daemon，就将进程设置为 daemon，这里判断如果是 MacOSX 就不做这个事情，因为 MacOSX 10.5 已经弃用了 Daemon，就直接写 pid 文件，关闭标准输出、输入、出错
- 初始化 harbor skynet_harbor_init 
- 初始化 handle skynet_handle_init(config->harbor);
- 初始化 MQ skynet_mq_init();
- 初始化 module skynet_module_init(config->module_path); 
- 初始化 timer skynet_timer_init(); 
- 初始化 Network skynet_socket_init();
- 初始化 profile skynet_profile_enable(config->profile); 
- 初始化 Log 对象和 bootstrap相关信息 
- 开始运行
	- 创建 Monitor 对象，用来给外部对整个系统的对 线程描述信息数组，锁， sleep， quit 等信息的跟踪。（这里是主线程）
	- 创建3个线程 和 pthread_mutex pthread_cond 每个线程包含的信息：版本 Mornitor ，check_version, source, destination，每个线程的这些信息放在 Mornitor 里面的线程描述信息组里。这三个线程分别是 Mornitor， Timer， socket 线程
	- 创建 12个线程，都是 thread_worker, 每个线程开始自己的工作
	- 每个 worker 线程开始自己的无限循环，从 MQ 中拿数据来做事情。这个动作都是用 mutex 锁定的保证线程安全。

这就是整个启动逻辑的全部过程，后期每个 worker 如何工作都是通过 MQ 的指令来做的。基本框架就出来了

## 环境变量部分

在 skynet 中环境变量定义的是在 skynet_env.c/h 中定义的，这部分简单就是利用一个 lua_State 放入一些 global 的变量，这些变量可以通过spinlock来限制线程同步获取和设置。本身才去了线程安全。
在环境变量的处理中有 
```C
void skynet_setenv(const char * key, const char * val); 
```
这其实也就是往所对应的 lua_State 里面放入一些变量存储而已，没有特别的。
```C
const char * skynet_getenv(const char * key);
```
就是读取存放在 lua_State 里面的变量返回出来

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

## Config 部分
这个系统采用 Lua 文件作为整个系统的配置。首先加载的过程中，启动代码中有一个字符串变量，包含的是初始化必备的 config 信息。
```C
static const char * load_config = "\
	local result = {}\n\
	local function getenv(name) return assert(os.getenv(name), [[os.getenv() failed: ]] .. name) end\n\
	local sep = package.config:sub(1,1)\n\
	local current_path = [[.]]..sep\n\
	local function include(filename)\n\
		local last_path = current_path\n\
		local path, name = filename:match([[(.*]]..sep..[[)(.*)$]])\n\
		if path then\n\
			if path:sub(1,1) == sep then	-- root\n\
				current_path = path\n\
			else\n\
				current_path = current_path .. path\n\
			end\n\
		else\n\
			name = filename\n\
		end\n\
		local f = assert(io.open(current_path .. name))\n\
		local code = assert(f:read [[*a]])\n\
		code = string.gsub(code, [[%$([%w_%d]+)]], getenv)\n\
		f:close()\n\
		assert(load(code,[[@]]..filename,[[t]],result))()\n\
		current_path = last_path\n\
	end\n\
	setmetatable(result, { __index = { include = include } })\n\
	local config_name = ...\n\
	include(config_name)\n\
	setmetatable(result, nil)\n\
	return result\n\
";
```

这段代码主要的内容是提供基本函数 getenv setenv include setmetatable 等基本的操作，为了在配置文件的设置和读取中有一定的帮助。
系统先创建一个 lua_State 加载这段字符串变成代码段，然后在这个 包含了上述功能的 lua_State 里再去将对应定义的内容一步一步的 set 到 skynet_env.h/c 模块的lua_State 中，这样让 skynet_env 这个模块就有上面字符串里包含的最基本的 get set include setmetatable 等功能了。这就是整个 config 模块的功能。简单说就是创建一个 setnet_env 的模块


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

```C
static inline int
fill_size(uint8_t * data, int sz) {
	data[0] = sz & 0xff;
	data[1] = (sz >> 8) & 0xff;
	data[2] = (sz >> 16) & 0xff;
	data[3] = (sz >> 24) & 0xff;
	return sz + SIZEOF_LENGTH;
}

static int
encode_integer(uint32_t v, uint8_t * data, int size) {
	if (size < SIZEOF_LENGTH + sizeof(v))
		return -1;
	data[4] = v & 0xff;
	data[5] = (v >> 8) & 0xff;
	data[6] = (v >> 16) & 0xff;
	data[7] = (v >> 24) & 0xff;
	return fill_size(data, sizeof(v));
}

static int encode_uint64(uint64_t v, uint8_t * data, int size) {
	if (size < SIZEOF_LENGTH + sizeof(v))
		return -1;
	data[4] = v & 0xff;
	data[5] = (v >> 8) & 0xff;
	data[6] = (v >> 16) & 0xff;
	data[7] = (v >> 24) & 0xff;
	data[8] = (v >> 32) & 0xff;
	data[9] = (v >> 40) & 0xff;
	data[10] = (v >> 48) & 0xff;
	data[11] = (v >> 56) & 0xff;
	return fill_size(data, sizeof(v));
}
```

这 encoder 的逻辑很神奇，把一个更大的空间的类型按照不同的 bit 给数据存放到比他更小空间的数组中。这样其实是可以保证一个我们看似会丢数据的逻辑，数据保持了完整性。
这里记录一下以后可能有很大的帮助。



## Network 线程
 基础的TCP/IP 接口的实现，支持 TCP UDP 的协议。对于 MacosX 和 FreeBSD 系统在编译的时候通过宏定义导向逻辑 KEvent 来实现(socket_server.c, socket_pool.h, socket_kqueue.h)，Linux 下面导向 epoll (socket_epoll.h)实现. 核心函数 send_request

 ```C
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

## SpinLock 自旋锁
这个项目中多线程对一个资源的加锁通常都使用的是 spinLock，在代码里面除了 spinLock 之外还有 mutex，但是好像没有看到有semaphore的加锁。
所有自旋锁的逻辑都放在 spinlock.h/c 中

这里 spinLock, mutex 和 Semaphore 有很大的区别，列一下：
- 工作方法：
	* SpinLock 处于忙等待的情况，就是如果尝试被锁的资源已经被锁定，就会一直尝试检查，直到被自己锁定位置
	* Mutex 线程尝试获取锁，如果已经被其他线程锁定，则线程会进入休眠（阻塞），当这个资源的锁被释放后，操作系统会唤醒被阻塞的锁来执行
	* Semaphore 通过信号量维护一个计数器，来控制多个线程多资源的访问，可以让多个线程同时访问一个被锁定的临界区，通常分为两种：binary Semaphore (BS) 和 counting Semaphore(CS). BS类似于Mutex 只有 0， 1， 允许访问和不允许。CS 计数器大于 1 的时候表示可以有多个线程访问，超过计数的 Count 之后，CS 计数器就为 0，这样就不能被访问了。这个复杂度比较高，适用于生产者消费者的模型。
- 优缺点：
	* SpinLock 在响应速度上很快，适合快速的场景，比如就是系统逻辑，没有 IO，网络等参与的逻辑，但是一直会占用 CPU 资源做无限循环，不适合对长时间的操作加锁
	* Mutex 节约CPU，适合在被锁的逻辑中有比较长时间的操作，比如 IO，网络，进程交互，大内存空间的扫描。因为线程要挂起，就会出现线程上下文切换的的问题，这就让这个操作效率比较低了
	* Semaphore 灵活性很高，可以针对多个线程访问资源的限制，不仅仅是 0 和 1 的关系。但是复杂性较高，可能出现死锁，在访问连接池，线程池，这类操作的时候比较适用

## MQ 线程
这里面自己实现了一套 MessageQueue 的逻辑，主要放在 skynet_mq.c/h 中，主要还是采用上面说的 SpinLock 保证线程安全。
```C
struct message_queue {
	struct spinlock lock;
	uint32_t handle;
	int cap;
	int head;
	int tail;
	int release;
	int in_global;
	int overload;
	int overload_threshold;
	struct skynet_message *queue;
	struct message_queue *next;
};

struct global_queue {
	struct message_queue *head;
	struct message_queue *tail;
	struct spinlock lock;
};
static struct global_queue *Q = NULL;
```
这里定义了专门的 Queue 和 message 对象。以 Q 为核心操作对象，包括有 push，pop，除了 Global Queue 之外还允许独立创建 Queue，但是需要创建者自己来管理，只是这个模块提供了 push pop 的操作而已。
整个 Queue 的内部采用循环队列来管理 all messages，当一个 queue 的循环队列满了，会自动expand_queue 这个队列，他的逻辑是 expand 当时队列的长度，就是将队列变成 2 倍那么长。
MQ 被使用的地方：

```
1. 多个线程之间数据交换的基本通道，为了保证线程安全，多个线程之间的MQ
2. 可以是多进程之间的信息传递，如果你配置了多进程的运行模式的话
3. 传递的数据就是Context， 中间包含一般的数据，也包含 Lua 的 Module instance， 这样可以做到不同 Lua 解释器之间的数据共享
```

## Module 
Module 管理层的逻辑可以帮助建立 Module Instance，所有建立的 Module Instance 都会在一个 
```C
static struct modules * M = NULL;
struct modules {
	int count;
	struct spinlock lock;
	const char * path;
	struct skynet_module m[MAX_MODULE_TYPE];
};//也就是说这个 Modules 最多管理MAX_MODULE_TYPE 个 m，MAX_MODULE_TYPE = 32 的，对过管理 32 个 Module
```
的指针中保存下来。这是 若干个 Module 的管理系统。

Module instance 创建过程：
skynet_module.c 中有这么个函数 

```C
static void * _try_open(struct modules *m, const char * name);
```

这个函数的意思就是打开一个名字叫做 name 的Module，他会去很多路径查询并且试图去加载，直到加载成功后返回 module 的指针（void *）
然后将 返回的 module instance 的指针放入前面说的struct skynet_module m[MAX_MODULE_TYPE];数组中。然后正式 open 这个 module instance，去找这个 module instance 的 api 是否存在_create, _init _release _signal, 分别赋值给 modules 结构提里面的函数指针。
下面是每个 Module instance的含义：
然后，每一个 Module Instance 都有这么几个函数： (前面是函数类型，后面是函数名)

```
	skynet_dl_create : create
	skynet_dl_init  : init
	skynet_dl_release : release
	skynet_dl_signal : signal
```

一个 Module 在创建的时候一定会调用 create，但是不会掉用 init，如果没有实现 create 函数，那么就会返回一个错误指针：0xFFFFFFFF，否则会返回 create 函数的返回值，
Module instance 是可以用他的name 来检索是否包含在 Modules 里面的
Module instance 创建完以后一般会包装成为一个 context， 然后通过handle_resister 注册到 handle 里面，这部分就涉及到 Handler 的讲解了

## Handler
本身 Handler 管理很多 Module 和其他相关对象的 Context。他也有自己的管理对象：
```C
struct handle_storage {
	struct rwlock lock;

	uint32_t harbor;
	uint32_t handle_index;
	int slot_size;
	struct skynet_context ** slot;
	
	int name_cap;
	int name_count;
	struct handle_name *name;
};

static struct handle_storage *H = NULL;
```
Handler 模块初始化的逻辑大概是给他的 H 创建内存，并且 H 中的 slot 也创建内存，

在这个模块里面他提供了 register, retire, retire_all, grab, find_name 等函数让别人去操作 handle 对象
具体 Handler 和 Module 之间的区别后面看懂了再在这里写，从现在读到的深度似乎发现了 Handler 比 Module更加 common 对的类。这可能是除了 Module 之外还有其他的东西需要通过 Handler 来操作，用  将 Context 包装一层 Module 放入 Hnadler 里面。（总之要确认才知道）

另外 handler 是也可以用名字检索

## Context 对象
上面说了 Module 会被包装成一个Context 放入 Handler，这个 Context 是个什么？来看 skynet_server.c 是怎么定义 context 的
```C
struct skynet_context {
	void * instance;
	struct skynet_module * mod;
	void * cb_ud;
	skynet_cb cb;
	struct message_queue *queue;
	ATOM_POINTER logfile;
	uint64_t cpu_cost;	// in microsec
	uint64_t cpu_start;	// in microsec
	char result[32];
	uint32_t handle;
	int session_id;
	ATOM_INT ref;
	size_t message_count;
	bool init;
	bool endless;
	bool profile;

	CHECKCALLING_DECL
};
```
里面包含 module， queue，的信息我所理解的为什么会包含 queue 是因为context 会放入一些queue， 这样的反向引用提高效率，否则要查询会非常麻烦。
中了我的猜想，原来 context 就是为了通过做MQ 进行传输数据时候使用的，也就是说多个线程或者进程之间的通信，数据都需要通过 context 来包装以后才能够传递，handler 要传递或者接收消息如果别人穿过来的是一个 module,这样也是可以的。这就是为什么 context 对象里面会包含
```C
struct skynet_module * mod;
```
字段。我所能想象到的就是当一个 Lua 对象在虚拟机里面，可以是一个 Module，然后通过这个Context 的封装，可以传递出去给另外的线程去处理，这样变相的实现了在 Lua不支持多线程的情况下用线程之间传递 Module 的方式来传递数据，也就可以让一个 Module 就保存好一个玩家专属的信息就好。这样就不会出现多线程访问 Lua 虚拟机导致crash 的问题。也就是这个 Skynet 的核心部分。将 Lua 的一部分，打包传递到其他线程去工作。然后可以传回来。

Context 除了支持传递 Module 之外还可以支持传递 common 的数据，

```C
static const char * cmd_timeout(struct skynet_context * context, const char * param);
static const char * cmd_signal(struct skynet_context * context, const char * param);
static const char * cmd_logoff(struct skynet_context * context, const char * param);
static const char * cmd_logon(struct skynet_context * context, const char * param);
static const char * cmd_stat(struct skynet_context * context, const char * param);
static const char * cmd_monitor(struct skynet_context * context, const char * param);
static const char * cmd_abort(struct skynet_context * context, const char * param); 
static const char * cmd_starttime(struct skynet_context * context, const char * param); 
static const char * cmd_setenv(struct skynet_context * context, const char * param);
static const char * cmd_getenv(struct skynet_context * context, const char * param);
static const char * cmd_launch(struct skynet_context * context, const char * param);
static const char * cmd_kill(struct skynet_context * context, const char * param);
static const char * cmd_exit(struct skynet_context * context, const char * param);
static const char * cmd_name(struct skynet_context * context, const char * param);
static const char * cmd_query(struct skynet_context * context, const char * param); 
static const char * cmd_reg(struct skynet_context * context, const char * param);

static struct command_func cmd_funcs[] = {
	{ "TIMEOUT", cmd_timeout },
	{ "REG", cmd_reg },
	{ "QUERY", cmd_query },
	{ "NAME", cmd_name },
	{ "EXIT", cmd_exit },
	{ "KILL", cmd_kill },
	{ "LAUNCH", cmd_launch },
	{ "GETENV", cmd_getenv },
	{ "SETENV", cmd_setenv },
	{ "STARTTIME", cmd_starttime },
	{ "ABORT", cmd_abort },
	{ "MONITOR", cmd_monitor },
	{ "STAT", cmd_stat },
	{ "LOGON", cmd_logon },
	{ "LOGOFF", cmd_logoff },
	{ "SIGNAL", cmd_signal },
	{ NULL, NULL },
};
```
这里一系列的函数定时是内部模块之间的通信所用的，主要以下面的 cmd_funcs 数组形式与字符串对应，相当于一个命令表，这样可以让进程见通行更加方便。在这个基础上外部只要向函数skynet_command 发送 Context 就能够让服务器在进程或者线程中间实现调用，具体的参数都放在 Context 里面，这里面也有上面所将的 Handler 和 Module 信息。会让系统更加统一的进行交互。

## Timer
skynet_timer.c/h
这里管理 Timer 逻辑比较复杂但是并不是很难懂，首先结构体定义：
```C
struct timer {
	struct link_list near[TIME_NEAR]; //256 个 link_list 每个 link_list 有多少个元素自己去挂上就可以了
	struct link_list t[4][TIME_LEVEL]; //4*64 个 link_list 也是 256 个,但是分成 4 份
	struct spinlock lock;
	uint32_t time;
	uint32_t starttime;
	uint64_t current; //代表现在有多少个 timer
	uint64_t current_point;
};
```
这里定义了timer 的基本属性，包括 starttime current 还有一个 sprinlock 用来处理线程锁的时候对时间的修改。

首先在 start 里面会起一个 timer 相关的线程，这个线程也是死循环，每次 skynet_updatetime, skynet_socket_updatetime.
```
skynet_updatetime:主要是更新普通的定时器，每次 check 所有定时器对象（link_list中的）时间到了或者大于了就去执行，否则就不执行并且更新其time
skynet_socket_updatetime: 每次只是将 socket 部分的时钟更新而已不存在逻辑调用
```

## Service 模块
这个模块放在与 skynet-src平级的目录 service-src 下。怎么理解 service 的含义，这其实就是逻辑上层的实现，但是又不能放在 Lua 中实现，因为对性能要求比较高，所以就用 C 来实现了。这部分的内容主要包括：
```
databuffer.h //主要是通信数据的管理，databuffer 的定义（环形链表节点），message（单链表节点） 的定义，messagepool的实现，支持多个messagepool 行程一个 messagepool_list(单链表)，在并发条件下如何处理数据的 push， pop， read，free， 详细的要说的话，数据分为头部和 body

hashid.h //使用单链表来解决 hash 冲突，实现了一个 hash 表（重复造轮子了）
service_gate.c //实现了一个网关服务，定义的 struct connection 就是一个一个的连接对象，针对这些连接端详的连接处理，和 message_dispatch,底层都是调用skynet_socket 部分的逻辑来实现的。
service_harbor.c //这里是定义了远程消息发送的接口，他叫做 Harbor 港口的意思，就是说一个 SkyNet 节点与另外一个节点之间的联系是依靠 Harbor 进行的。从他的介绍上来看他并不赞成使用多服结构，多着多进程结构，他希望的是在他的 SkyNet 进程中用不同的线程去完成所有的事情，但是他也提供了一个能够向外扩展的多进程或者跨服的接口，这就是。
service_logger.c //一个提供了写入 log 信息的函数集
service_snlua.c //这个模块就是实现了初始化 Lua 环境，并且加载执行 Lua 脚本，协程调度和内存管理的模块，包括实现了协程调度，信号处理，属于 Lua 的标准库扩展，性能分析，等等。
```
如果要详细分析这些，需要很多时间，现在只拿到的是一个概括而已。

## Monitor 线程
这个进程的事情很简单，首先管理的事情就 version check_version， source， destination
```C
struct skynet_monitor {
	ATOM_INT version;
	int check_version;
	uint32_t source;
	uint32_t destination;
};
```
这个模块的初始化函数skynet_monitor_new，是在skynet_start start 函数中， 有多少个 thread 就会创建多少个 Monitor 对象，然后放在数组里。
```C
struct monitor *m = skynet_malloc(sizeof(*m));
for (i=0;i<thread;i++) {
		m->m[i] = skynet_monitor_new();//create an instance of monitor 
}
```
然后创建一个 monitor 线程, 线程所做的事情就是一个死循环，不听的监控m中的每一个元素的状态，逻辑也比较简单，检测version 和 check_version 是否相等，不等就设置相等，如果 destination 有问题就回收内存，并且报错，让他退出


```C 参考
static unsigned int HARBOR = ~0; //按位取反 结果是 0xFFFFFFFF 这是获取最大 unsigned int 的最好办法
```



## Lua 代码部分 (./lualib )
请查看这个文件：[NOTE_LUA.md](./NOTE_LUA.md)

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

#### .PHONY
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


## 读代码的总结

读代码不能一字一句的去读，一段一段的读，明白他这段是什么意思，然后读下一段，不能跳进去精细的去读。要保持脑子里的那个框架去读。框架整理清楚了以后，每个模块再去细读，这样才能弄清楚他的思想和做法。

以前读大话 3 服务器段引擎我完全是一句一句去读的，因为他简单所以能读懂。这种复杂的项目一句一句去读，马上脑子就死掉了读不下去了，

