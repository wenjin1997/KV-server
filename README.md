# KV-server
客户端通过网络访问 KV server，发送包含命令的请求。服务端在接收到命令后，根据不同的命令进行数据存储、读取、监听等操作，将结果返回给客户端。

## 使用
客户端：
```rust
use anyhow::Result;
use kv::{CommandRequest, ProstClientStream, TlsClientConnector};
use tokio::net::TcpStream;
use tracing::info;

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt::init();

    // 可用配置替换
    let ca_cert = include_str!("../../fixtures/ca.cert");

    let addr = "127.0.0.1:9527";
    // 连接服务器
    let connector = TlsClientConnector::new("kvserver.acme.inc", None, Some(ca_cert))?;
    let stream = TcpStream::connect(addr).await?;
    let stream = connector.connect(stream).await?;

    let mut client = ProstClientStream::new(stream);

    // 生成一个 HSET 命令
    let cmd = CommandRequest::new_hset("table1", "hello", "world".to_string().into());

    // 发送 HSET 命令
    let data = client.execute(cmd).await?;
    info!("Got response {:?}", data);

    Ok(())
}
```

服务端：
```rust
use anyhow::Result;
use kv::{MemTable, ProstServerStream, Service, ServiceInner, TlsServerAcceptor};
use tokio::net::TcpListener;
use tracing::info;

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt::init();
    let addr = "127.0.0.1:9527";

    // 可从配置文件取
    let server_cert = include_str!("../../fixtures/server.cert");
    let server_key = include_str!("../../fixtures/server.key");

    let acceptor = TlsServerAcceptor::new(server_cert, server_key, None)?;
    let service: Service = ServiceInner::new(MemTable::new()).into();
    let listener = TcpListener::bind(addr).await?;
    info!("Start listening on {}", addr);
    loop {
        let tls = acceptor.clone();
        let (stream, addr) = listener.accept().await?;
        info!("Client {:?} connected", addr);
        let stream = tls.accept(stream).await?;
        let stream = ProstServerStream::new(stream, service.clone());
        tokio::spawn(async move { stream.process().await });
    }
}
```

运行命令：

```
RUST_LOG=info cargo run --bin server --quiet
RUST_LOG=info cargo run --bin client --quiet
```