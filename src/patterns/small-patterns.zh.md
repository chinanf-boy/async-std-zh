# Small Patterns

小而有用的模式的集合。

## Splitting streams

`async-std`没有提供`split()`方法开启`io`处理。相反，可以将流分成读写部分，如下所示：

```rust,edition2018
# extern crate async_std;
use async_std::{io, net::TcpStream};
async fn echo(stream: TcpStream) {
    let (reader, writer) = &mut (&stream, &stream);
    io::copy(reader, writer).await;
}
```
