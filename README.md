#   KV-server

## 需求

* 服务端能根据不同的命令进行诸如数据存贮、读取、监听等操作；
* 客户端要能通过网络访问 KV server，发送包含命令的请求，得到结果；
* 数据要能根据需要，存储在内存中或者持久化到磁盘上。

## 架构设计
![design](img/design.png)

1. 客户端和服务器之间采用的协议要灵活，先考虑TCP协议。
2. 客户端和服务器之间交互的应用层协议用protobuf定义。protobuf解决了协议的定义以及序列化和反序列化。同时protobuf解析效率高。
3. 服务器支持一些Redis命令，例如HSET、HMSET、HGET、HMGET等。
4. 处理流程中可以加入一些hook，方便调用者在初始化服务的时候注入额外的处理逻辑。
5. 存储方面支持内存存储、也支持持久化的存储。
   
## 主体交互接口
### 客户端和服务器
使用protobuf定义。在根目录下创建[abi.proto](/kv/abi.proto)，再在根目录下创建[build.rs](/kv/build.rs)。运行`cargo build`会生成[abi.rs](kv/src/pb/abi.rs)文件。

```rust
fn main() {
    let mut config = prost_build::Config::new();
    config.bytes(&["."]);
    config.type_attribute(".", "#[derive(PartialOrd)]");
    config
        .out_dir("src/pb")
        .compile_protos(&["abi.proto"], &["."])
        .unwrap();
}
```

接着在[src/pb/mod.rs](/kv/src/pb/mod.rs)中为CommandRequest、Kvpair实现一些方法，主要是一些命令的方法。

### CommandService

定义一个trait来统一处理所有的命令，返回处理结果。

### 存储
为了实现能支持不同的存储，可以设计一个 Storage trait，它提供 KV store 主要的接口。

## 错误处理
为了方便错误类型转换，定义一个KvError，用thiserror库来处理。代码见[error.rs](/kv/src/error.rs)。