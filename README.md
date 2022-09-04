- [KV-server](#kv-server)
  - [需求](#需求)
  - [架构设计](#架构设计)
  - [主体交互接口](#主体交互接口)
    - [客户端和服务器](#客户端和服务器)
    - [CommandService](#commandservice)
    - [存储](#存储)
      - [MemTable](#memtable)
      - [持久化数据库](#持久化数据库)
  - [错误处理](#错误处理)
  - [日志](#日志)
  - [Server](#server)
  - [处理 Iterator](#处理-iterator)
  - [支持事件通知](#支持事件通知)
  - [定义协议的Frame](#定义协议的frame)
  - [网络安全](#网络安全)
    - [生成x509证书](#生成x509证书)
    - [使用TLS](#使用tls)
  - [异步处理](#异步处理)

# KV-server
## 需求
KV server 的主要需求如下：
* 核心功能是服务端能根据不同的命令进行诸如数据存贮、读取、监听等操作；
* 客户端要能通过网络访问 KV server，发送包含命令的请求，得到结果；
* 数据要能根据需要，存储在内存中或者持久化到磁盘上。

## 架构设计
![design](img/design.png)

1. 客户端和服务器之间采用的协议要灵活，先考虑TCP协议。网络层需要灵活，后序为保证安全可以加上TLS协议。
2. 客户端和服务器之间交互的应用层协议用protobuf定义。protobuf解决了协议的定义以及序列化和反序列化。同时protobuf解析效率高。
3. 服务器支持一些Redis命令，例如HSET、HMSET、HGET、HMGET等。从命令到命令的响应，可以做个trait进行抽象。
4. 处理流程中可以加入一些hook，具体的hook有：收到客户端的命令后 OnRequestReceived、处理完客户端的命令后 OnRequestExecuted、发送响应之前 BeforeResponseSend、发送响应之后 AfterResponseSend。这样可以方便调用者在初始化服务的时候注入额外的处理逻辑。
5. 存储方面需要灵活，可以对存储做个trait来抽象其基本行为，可以支持内存存储、也支持持久化的存储。
   
## 主体交互接口
### 客户端和服务器
使用protobuf定义，在根目录下创建[abi.proto](/kv/abi.proto)，主要定义`CommandRequest`以及`CommandResponse`。

```proto
// 来自客户端的命令请求
message CommandRequest {
  oneof request_data {
    Hget hget = 1;
    Hgetall hgetall = 2;
    Hmget hmget = 3;
    Hset hset = 4;
    Hmset hmset = 5;
    Hdel hdel = 6;
    Hmdel hmdel = 7;
    Hexist hexist = 8;
    Hmexist hmexist = 9;
  }
}

// 服务器的响应
message CommandResponse {
  // 状态码；复用 HTTP 2xx/4xx/5xx 状态码
  uint32 status = 1;
  // 如果不是 2xx，message 里包含详细的信息
  string message = 2;
  // 成功返回的 values
  repeated Value values = 3;
  // 成功返回的 kv pairs
  repeated Kvpair pairs = 4;
}
```

再在根目录下创建[build.rs](/kv/build.rs)。

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
这里为了将 protobuf 文件编译成 Rust 代码，使用了第三方库[prost](https://github.com/tokio-rs/prost)。在[Cargo.toml](/kv/Cargo.toml)的配置如下：

```toml
[dependencies]
prost = "0.9" # 处理 protobuf 的代码

[build-dependencies]
prost-build = "0.9" # 编译 protobuf
```

运行`cargo build`会生成[abi.rs](kv/src/pb/abi.rs)文件。

接着在[src/pb/mod.rs](/kv/src/pb/mod.rs)中为CommandRequest实现一些方便的命令方法，以及对里面的数据实现一些转换方法，方便调用。

### CommandService
下面考虑如何处理请求的命令，返回响应。为了支持多种命令，考虑到后序的扩展，可以定义一个trait来统一处理所有的命令，返回处理结果。

创建[src/service/mod.rs](/kv/src/service/mod.rs)，在其中定义`CommandService` trait，对 Command 进行抽象。

```rust
/// 对 Command 的处理的抽象
pub trait CommandService {
    /// 处理 Command，返回 Response
    fn execute(self, store: &impl Storage) -> CommandResponse;
}
```

这样，对每个命令，具体要执行什么处理逻辑，就可以用一个函数来实现分发。

```rust
// 从 Request 中得到 Response，目前处理 HGET/HGETALL/HSET
pub fn dispatch(cmd: CommandRequest, store: &impl Storage) -> CommandResponse {
    match cmd.request_data {
        Some(RequestData::Hget(param)) => param.execute(store),
        Some(RequestData::Hgetall(param)) => param.execute(store),
        Some(RequestData::Hset(param)) => param.execute(store),
        None => KvError::InvalidCommand("Request has no data".into()).into(),
        _ => KvError::Internal("Not implemented".into()).into(),
    }
}
```

想要支持的命令可以为其实现 `CommandService` trait，以及在 dispatch 方法中加入命令的支持，这部分代码见 [src/service/command_service.rs](/kv/src/service/command_service.rs)。例如为 Hget 实现 CommandService。

```rust
impl CommandService for Hget {
    fn execute(self, store: &impl Storage) -> CommandResponse {
        match store.get(&self.table, &self.key) {
            Ok(Some(v)) => v.into(),
            Ok(None) => KvError::NotFound(self.table, self.key).into(),
            Err(e) => e.into(),
        }
    }
}
```
这里用到了 `v.into()`，可以在[src/pb/mod.rs](/kv/src/pb/mod.rs)中为响应的数据类型实现`From` trait，方便转换成 CommandResponse。例如从 `Value` 转换成 `CommandResponse`。

```rust
/// 从 Value 转换成 CommandResponse
impl From<Value> for CommandResponse {
    fn from(v: Value) -> Self {
        Self {
            status: StatusCode::OK.as_u16() as _,
            values: vec![v],
            ..Default::default()
        }
    }
}
```
### 存储
为了实现能支持不同的存储，可以设计一个 Storage trait，它提供 KV store 主要的接口。创建[src/storage/mod.rs](/kv/src/storage/mod.rs)。

```rust
/// 对存储的抽象，我们不关心数据存在哪儿，但需要定义外界如何和存储打交道
pub trait Storage {
    /// 从一个 HashTable 里获取一个 key 的 value
    fn get(&self, table: &str, key: &str) -> Result<Option<Value>, KvError>;
    /// 从一个 HashTable 里设置一个 key 的 value，返回旧的 value
    fn set(
        &self,
        table: &str,
        key: impl Into<String>,
        value: impl Into<Value>,
    ) -> Result<Option<Value>, KvError>;
    /// 查看 HashTable 中是否有 key
    fn contains(&self, table: &str, key: &str) -> Result<bool, KvError>;
    /// 从 HashTable 中删除一个 key
    fn del(&self, table: &str, key: &str) -> Result<Option<Value>, KvError>;
    /// 遍历 HashTable，返回所有 kv pair（这个接口不好）
    fn get_all(&self, table: &str) -> Result<Vec<Kvpair>, KvError>;
    /// 遍历 HashTable，返回 kv pair 的 Iterator
    fn get_iter(&self, table: &str) -> Result<Box<dyn Iterator<Item = Kvpair>>, KvError>;
}
```

这样，后期如果要添加不同的 store，只需要为其实现 Storage trait 即可，不必修改 CommandService 相关的代码。为了在多线程/异步环境下读取和更新，接口中的是 `&self` 参数。

#### MemTable
使用[dashmap](https://docs.rs/dashmap/latest/dashmap/index.html)来创建一个 MemTable 结构，`DashMap`结实现了并发。接口类似于HashMap，可看作`RwLock<HashMap<K, V, S>>`。创建文件[src/storage/memory.rs](/kv/src/storage/memory.rs)，为其实现 Storage trait。

```rust
/// 使用 DashMap 构建的 MemTable，实现了 Storage trait
#[derive(Clone, Debug, Default)]
pub struct MemTable {
    tables: DashMap<String, DashMap<String, Value>>,
}
```

#### 持久化数据库
使用[sled](https://github.com/spacejam/sled)库来实现持久化数据库的支持。
> sled is a high-performance embedded database with an API that is similar to a BTreeMap<[u8], [u8]>, but with several additional capabilities for assisting creators of stateful systems.
> It is fully thread-safe, and all operations are atomic. Multiple Trees with isolated keyspaces are supported with the Db::open_tree method.
> ACID transactions involving reads and writes to multiple items are supported with the Tree::transaction method. Transactions may also operate over multiple Trees (see Tree::transaction docs for more info).

创建[src/storage/sleddb.rs](kv/src/storage/sleddb.rs)并实现Storage trait。

## 错误处理
为了方便错误类型转换，定义一个KvError，用[thiserror](https://github.com/dtolnay/thiserror)派生宏来定义错误类型。代码见[error.rs](/kv/src/error.rs)。

## 日志
使用tracing与tracing-subscriber进行日志处理。日志处理例子见[examples/server.rs](kv/examples/server.rs)。

## Server
在[src/service/mod.rs](/kv/src/service/mod.rs)中添加 `Service` 结构。

```rust
/// Service 数据结构
pub struct Service<Store = MemTable> {
    inner: Arc<ServiceInner<Store>>,
}

impl<Store> Clone for Service<Store> {
    fn clone(&self) -> Self {
        Self {
            inner: Arc::clone(&self.inner),
        }
    }
}

/// Service 内部数据结构
pub struct ServiceInner<Store> {
    store: Store,
}

impl<Store: Storage> Service<Store> {
    pub fn new(store: Store) -> Self {
        Self {
            inner: Arc::new(ServiceInner { store }),
        }
    }

    pub fn execute(&self, cmd: CommandRequest) -> CommandResponse {
        debug!("Got request: {:?}", cmd);
        // TODO: 发送 on_received 事件
        let res = dispatch(cmd, &self.inner.store);
        debug!("Executed response: {:?}", res);
        // TODO: 发送 on_executed 事件

        res
    }
}

// 从 Request 中得到 Response，目前处理 HGET/HGETALL/HSET
pub fn dispatch(cmd: CommandRequest, store: &impl Storage) -> CommandResponse {
    match cmd.request_data {
        Some(RequestData::Hget(param)) => param.execute(store),
        Some(RequestData::Hgetall(param)) => param.execute(store),
        Some(RequestData::Hset(param)) => param.execute(store),
        None => KvError::InvalidCommand("Request has no data".into()).into(),
        _ => KvError::Internal("Not implemented".into()).into(),
    }
}
```

1. Service 结构内部是 ServiceInner 存放实际的数据结构，Service 对其用 Arc 包裹，这样的话就可以在多线程下把 clone 的主体和其内部结构分开，代码逻辑更加清晰。
2. execute() 方法后面还可以实现一些事件的分发。

## 处理 Iterator
想要为每个 Storage trait 实现 get_iter() 方法。可以使用 IntoIterator trait。

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```
绝大多数数据结构都实现了它，DashMap也实现了这个 trait。这样MemTable 就可以这样实现 get_iter() 方法了。

```rust
impl Storage for MemTable {
    ...
    fn get_iter(&self, table: &str) -> Result<Box<dyn Iterator<Item = Kvpair>>, KvError> {
        // 使用 clone() 来获取 table 的 snapshot
        let table = self.get_or_create_table(table).clone();
        let iter = table.into_iter().map(|data| data.into());
        Ok(Box::new(iter))
    }
}
```
这里有`data.into()`，我们可以为 `Kvpair` 实现 `From` trait。

```rust
impl From<(String, Value)> for Kvpair {
    fn from(data: (String, Value)) -> Self {
        Kvpair::new(data.0, data.1)
    }
}
```

这里一个 store 处理 get_iter() 方法的流程是：
1. 拿到一个关于某个 table 下的拥有所有权的 Iterator；
2. 对 Iterator 做 map；
3. 将 map 出来的每个 item 转换成 Kvpair。

我们可以对第2步进行封装。在[/src/storage/mod.rs](/kv/src/storage/mod.rs)中构建一个 `StorageIter`，然后为其实现 `Iterator` trait。

```rust
/// 提供 Storage iterator，这样 trait 的实现者只需要
/// 把它们的 iterator 提供给 StorageIter，然后它们保证
/// next() 传出的类型实现了 Into<Kvpair> 即可
pub struct StorageIter<T> {
    data: T,
}

impl<T> StorageIter<T> {
    pub fn new(data: T) -> Self {
        Self { data }
    }
}

impl<T> Iterator for StorageIter<T>
where
    T: Iterator,
    T::Item: Into<Kvpair>,
{
    type Item = Kvpair;

    fn next(&mut self) -> Option<Self::Item> {
        self.data.next().map(|v| v.into())
    }
}
```

这样，原来 MemTable 的 get_iter() 方法就变成了这样。

```rust
impl Storage for MemTable {
    ...
    fn get_iter(&self, table: &str) -> Result<Box<dyn Iterator<Item = Kvpair>>, KvError> {
        // 使用 clone() 来获取 table 的 snapshot
          let table = self.get_or_create_table(table).clone();
          let iter = StorageIter::new(table.into_iter()); // 这行改掉了
          Ok(Box::new(iter))
      }
}
```

## 支持事件通知
事件通知机制：
1. 在创建 Service 时，注册相应的事件处理函数；
2. 在 execute() 方法执行时，做相应的事件通知，使得注册的事件处理函数得到执行。

设计了四个事件：
1. on_received：当服务器收到 CommandRequest 时触发；
2. on_executed：当服务器处理完 CommandRequest 得到 CommandResponse 时触发；
3. on_before_send：在服务器发送 CommandResponse 之前触发。注意这个接口提供的是 &mut CommandResponse，这样事件的处理者可以根据需要，在发送前，修改 CommandResponse。
4. on_after_send：在服务器发送完 CommandResponse 后触发。

```rust
/// Service 内部数据结构
pub struct ServiceInner<Store> {
    store: Store,
    on_received: Vec<fn(&CommandRequest)>,
    on_executed: Vec<fn(&CommandResponse)>,
    on_before_send: Vec<fn(&mut CommandResponse)>,
    on_after_send: Vec<fn()>,
}
```

为了调用者方便注册事件，使用链式调用，我们可以为ServiceInner实现如下的方法，具体可见[src/service/mod.rs](/kv/src/service/mod.rs)。

```rust
impl<Store: Storage> ServiceInner<Store> {
    pub fn new(store: Store) -> Self {
        Self {
            store,
            on_received: Vec::new(),
            on_executed: Vec::new(),
            on_before_send: Vec::new(),
            on_after_send: Vec::new(),
        }
    }

    pub fn fn_received(mut self, f: fn(&CommandRequest)) -> Self {
        self.on_received.push(f);
        self
    }

    pub fn fn_executed(mut self, f: fn(&CommandResponse)) -> Self {
        self.on_executed.push(f);
        self
    }

    pub fn fn_before_send(mut self, f: fn(&mut CommandResponse)) -> Self {
        self.on_before_send.push(f);
        self
    }

    pub fn fn_after_send(mut self, f: fn()) -> Self {
        self.on_after_send.push(f);
        self
    }
}
```

下面实现事件的通知:

```rust
/// 事件通知（不可变事件）
pub trait Notify<Arg> {
    fn notify(&self, arg: &Arg);
}

/// 事件通知（可变事件）
pub trait NotifyMut<Arg> {
    fn notify(&self, arg: &mut Arg);
}


impl<Arg> Notify<Arg> for Vec<fn(&Arg)> {
    #[inline]
    fn notify(&self, arg: &Arg) {
        for f in self {
            f(arg)
        }
    }
}

impl<Arg> NotifyMut<Arg> for Vec<fn(&mut Arg)> {
  #[inline]
    fn notify(&self, arg: &mut Arg) {
        for f in self {
            f(arg)
        }
    }
}
```

至此，Service的execute方法就可以这样写了。

```rust
impl<Store: Storage> Service<Store> {
    pub fn execute(&self, cmd: CommandRequest) -> CommandResponse {
        debug!("Got request: {:?}", cmd);
        self.inner.on_received.notify(&cmd);
        let mut res = dispatch(cmd, &self.inner.store);
        debug!("Executed response: {:?}", res);
        self.inner.on_executed.notify(&res);
        self.inner.on_before_send.notify(&mut res);
        if !self.inner.on_before_send.is_empty() {
            debug!("Modified response: {:?}", res);
        }

        res
    }
}
```

## 定义协议的Frame
要区分不同的消息可以用\r\n这种进行分隔符，也可以用长度进行分隔。由于KV server使用的是protobuf来定义协议，承载的是二进制，因此在 payload 之前放一个长度，来作为 frame 的分隔。

使用 tokio-util 中的 [LengthDelimitedCodec](https://docs.rs/tokio-util/latest/tokio_util/codec/length_delimited/index.html)。见[examples/server_with_codec.rs](/kv/examples/server_with_codec.rs)。

用下面的命令运行：
```bash
RUST_LOG=info cargo run --example server_with_codec --quiet
RUST_LOG=info cargo run --example client --quiet
```

自己处理Frame，可以设置长度为4个字节，这样payload可以到4G。考虑到压缩，把4字节长度的最高位拿出来作为是否压缩的信号。

![frame](img/frame.png)

定义 FrameCoder trait。见[src/network/frame.rs](/kv/src/network/frame.rs)。

```rust
pub trait FrameCoder
where
    Self: Message + Sized + Default,
{
    /// 把一个 Message encode 成一个 frame
    fn encode_frame(&self, buf: &mut BytesMut) -> Result<(), KvError>;
    /// 把一个完整的 frame decode 成一个 Message
    fn decode_frame(buf: &mut BytesMut) -> Result<Self, KvError>;
}
```
* encode_frame() 可以把诸如 CommandRequest 这样的消息封装成一个 frame，写入传进来的 BytesMut；
* decode_frame() 可以把收到的一个完整的、放在 BytesMut 中的数据，解封装成诸如 CommandRequest 这样的消息。

如果要实现这个 trait，Self 需要实现了 prost::Message，大小是固定的，并且实现了 Default（prost 的需求）。

使用[flate2](https://github.com/rust-lang/flate2-rs)库处理gzip压缩。具体实现代码见[src/network/frame.rs](kv/src/network/frame.rs)。

这里设置COMPRESS_LIMIT位1436。因为以太网的 MTU 是 1500，除去 IP 头 20 字节、TCP 头 20 字节，还剩 1460；一般 TCP 包会包含一些 Option（比如 timestamp），IP 包也可能包含，所以我们预留 20 字节；再减去 4 字节的长度，就是 1436，不用分片的最大消息长度。如果大于这个，很可能会导致分片，我们就干脆压缩一下。

decode_frame() 函数使用BytesMut，还需要处理从 socket 中拿出来。先读4个字节，取出长度 N，然后再读 N 个字节。因此写个辅助函数 read_frame()。

```rust
/// 从 stream 中读取一个完整的 frame
pub async fn read_frame<S>(stream: &mut S, buf: &mut BytesMut) -> Result<(), KvError>
where
    S: AsyncRead + Unpin + Send,
{
    let header = stream.read_u32().await? as usize;
    let (len, _compressed) = decode_header(header);
    // 如果没有这么大的内存，就分配至少一个 frame 的内存，保证它可用
    buf.reserve(LEN_LEN + len);
    buf.put_u32(header as _);
    // advance_mut 是 unsafe 的原因是，从当前位置 pos 到 pos + len，
    // 这段内存目前没有初始化。我们就是为了 reserve 这段内存，然后从 stream
    // 里读取，读取完，它就是初始化的。所以，我们这么用是安全的
    unsafe { buf.advance_mut(len) };
    stream.read_exact(&mut buf[LEN_LEN..]).await?;
    Ok(())
}
```
要求泛型S满足 AsyncRead trait，是 tokio 下的一个 trait，用于异步读取。

```rust
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        buf: &mut ReadBuf<'_>
    ) -> Poll<Result<()>>;
}
```

一旦某个数据结构实现了 AsyncRead，它就可以使用 AsyncReadExt 提供的多达 29 个辅助方法。这是因为任何实现了 AsyncRead 的数据结构，都自动实现了 AsyncReadExt：

```rust
impl<R: AsyncRead + ?Sized> AsyncReadExt for R {}
```

对于 Socket 来说，读取 socket 就是一个不断 poll_read() 的过程，直到读到了满足 ReadBuf 需要的内容。

为了让客户端和服务端更方便地对流进行处理，在网络层再进行一层封装，方便后续支持更多的功能。见[src/network/mod.rs](/kv/src/network/mod.rs)。

```rust
/// 处理服务器端的某个 accept 下来的 socket 的读写
pub struct ProstServerStream<S> {
    inner: S,
    service: Service,
}

/// 处理客户端 socket 的读写
pub struct ProstClientStream<S> {
    inner: S,
}
```
这里 S 是泛型参数，未来方便支持 WebSocket，或者在 TCP 之上支持 TLS。后面再为服务器实现 process()， 为客户端实现 execute() 方法。

客户端的使用见[src/bin/client.rs](kv/src/bin/client.rs)，服务端使用见[src/bin/server.rs](kv/src/bin/server.rs)。

## 网络安全
使用TLS保证网络的安全性。

### 生成x509证书
TLS需要x509证书让客户端验证服务器是否是一个受信的服务器，甚至服务器验证客户端，确认对方是一个受信的客户端。

使用[certify](https://github.com/tyrchen/certify)库来生成各种证书。

在根目录下创建 fixtures 目录用来存放证书，创建[examples/gen_cert.rs](kv/examples/gen_cert.rs)，先生成一个 CA 证书，然后再生成服务器和客户端证书，全部存入 fixtures 目录下。

### 使用TLS
TLS构建于TCP之上，对于 KV server，使用 TLS 之后，整个协议的数据封装如下图所示：

![data](img/data.png)

使用[tokio-rustls](https://github.com/tokio-rs/tls/tree/master/tokio-rustls)来支持 TLS。

创建[src/network/tls.rs](kv/src/network/tls.rs)文件，创建两个数据结构 `TlsServerAcceptor` 和 `TlsClientConnector`，根据提供的证书，来生成 tokio-tls 需要的 ServerConfig 和 ClientConfig。

再修改[src/bin/server.rs](kv/src/bin/server.rs)与[src/bin/client.rs](kv/src/bin/client.rs)让它们支持TLS。

## 异步处理
从收包处理到处理完后发包的完整流程和系统结构如下图所示：

![async](img/async.png)

对比[src/service/mod.rs](kv/src/service/mod.rs)中的ProstServerStream 的 process() 与 async_prost 库中 AsyncProst 的调用逻辑：

```rust
// process() 函数的内在逻辑
while let Ok(cmd) = self.recv().await {
    info!("Got a new command: {:?}", cmd);
    let res = self.service.execute(cmd);
    self.send(res).await?;
}

// async_prost 库的 AsyncProst 的调用逻辑
while let Some(Ok(cmd)) = stream.next().await {
    info!("Got a new command: {:?}", cmd);
    let res = svc.execute(cmd);
    stream.send(res).await.unwrap();
}
```

AsyncProst 实现了 Stream 和 Sink trait，因此可以自然地调用 StreamExt trait 的 next() 方法和 SinkExt trait 的 send() 方法，来处理数据的收发，而 ProstServerStream 则自己额外实现了函数 recv() 和 send()。

下面为了未来的可扩展性，和整个异步生态更加融洽，构造一个 ProstStream，实现 Stream 和 Sink 这两个 trait，然后让 ProstServerStream 和 ProstClientStream 使用它。

Stream trait 和 Sink trait 的定义：

```rust
// 可以类比 Iterator
pub trait Stream {
    // 从 Stream 中读取到的数据类型
    type Item;

    // 从 stream 里读取下一个数据
    fn poll_next(
    self: Pin<&mut Self>, cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;
}

// 
pub trait Sink<Item> {
    type Error;
    fn poll_ready(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;
    fn start_send(self: Pin<&mut Self>, item: Item) -> Result<(), Self::Error>;
    fn poll_flush(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;
    fn poll_close(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;
}
```

ProstStream 的具体实现见[src/network/stream.rs](kv/src/network/stream.rs)。先定义 ProstStream 结构，接着其实现 Stream 和 Sink trait。

```rust
/// 处理 KV server prost frame 的 stream
pub struct ProstStream<S, In, Out> where {
    // innner stream
    stream: S,
    // 写缓存
    wbuf: BytesMut,
    // 读缓存
    rbuf: BytesMut,

    // 类型占位符
    _in: PhantomData<In>,
    _out: PhantomData<Out>,
}
```

实现 ProstStream 后就可以修改 ProstServerStream 和 ProstClientStream 结构，具体代码见[src/network/mod.rs](/kv/src/network/mod.rs)。

```rust
/// 处理服务器端的某个 accept 下来的 socket 的读写
// // 旧的接口
// pub struct ProstServerStream<S> {
//     inner: S,
//     service: Service,
// }
pub struct ProstServerStream<S> {
    inner: ProstStream<S, CommandRequest, CommandResponse>,
    service: Service,
}

/// 处理客户端 socket 的读写
// // 旧的接口
// pub struct ProstClientStream<S> {
//     inner: S,
// }
pub struct ProstClientStream<S> {
    inner: ProstStream<S, CommandResponse, CommandRequest>,
}
```

最后在修改它们的一些实现方法。