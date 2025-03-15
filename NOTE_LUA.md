# SkyNet 引擎的Lua脚本部分

## 主要框架

### Project Structure
    lualib/
        ├── compat10/ # 这个目录是就的版本，在这里就不用说了
        │   ├── cluster.lua
        │   └── crypt.lua
        |   └── datacenter.lua
        |   └── dns.lua
        |   └── memory.lua
        |   └── mongo.lua
        |   └── mqueue.lua
        |   └── multicast.lua
        |   └── mysql.lua
        |   └── netpack.lua
        |   └── profile.lua
        |   └── redis.lua
        |   └── sharedata.lua
        |   └── sharemap.lua
        |   └── snax.lua
        |   └── socket.lua
        |   └── socketchannel.lua
        |   └── socketdriver.lua
        |   └── crypt.lua
        ├── http/ # http服务的逻辑，包括客户端，服务器端
        |   └── httpc.lua # HTTP 客户端的访问
        |   └── httpd.lua # HTTP 服务器端
        |   └── internal.lua #HTTP 服务的内部对 HTML 文件的获取和解析，从头分析到 body 读取分析，以及支持字节流的处理
        |   └── sockethelper.lua #对http创建通信等行为做tcp层面的辅助， 用的是 Skynet 的网络部分
        |   └── tlshelper.lua #TLS 对应的处理，就是 传统的 SSL 的处理，证书验证，等等加密通道
        |   └── url.lua # URL相关的处理，比如 decode， encode，字节处理等等
        |   └── websocket.lua # Websocket 的处理，在 HTTP 协议上层建立类似 TCP 的互相通信的渠道
        ├── skynet/ # 这里是 skynet 的核心区域
        |   └── datasheet\
        |   |   └── builder.lua 
        |   |   └── dump.lua
        |   |   └── init.lua
        |   └── db\ #对于 DB 的支持
        |   |   └── redis\ # redis的支持
        |   |   |   └── cluster.lua
        |   |   |   └── crc16.lua   #crc16 算法，用于cluster中的运算
        |   |   └── mongo.lua #mongodb的支持
        |   |   └── mysql.lua # mysql db的支持
        |   |   └── redis.lua # redis 支持的对外接口
        |   └── sharedata\
        |   |   └── corelib.lua
        |   └── cluster.lua
        |   └── coroutine.lua
        |   └── datacenter.lua
        |   └── debug.lua
        |   └── dns.lua
        |   └── harbor.lua
        |   └── inject.lua
        |   └── indectcode.lua
        |   └── manager.lua
        |   └── mqueue.lua
        |   └── multicast.lua
        |   └── queue.lua
        |   └── remotedebug.lua
        |   └── require.lua
        |   └── service.lua
        |   └── sharedatalua
        |   └── sharemap.lua
        |   └── sharetable.lua
        |   └── snax.lua
        |   └── socket.lua
        |   └── socketchannel.lua
        ├── snax/
        |   └── gateserver.lua
        |   └── hotfix.lua
        |   └── interface.lua
        |   └── logserver.lua
        |   └── msgserver.lua
        ├── loader.lua
        ├── md5.lua
        ├── skynet.lua
        ├── sproto.lua
        ├── sprotoloader.lua
        ├── sprotoparser.lua
        ├── .gitignore
        └── requirements.txt
