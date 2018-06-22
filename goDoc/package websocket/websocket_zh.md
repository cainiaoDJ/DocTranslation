# package websocket
import "github.com/gorilla/websocket"

websocket包是基于[RFC 6455](https://tools.ietf.org/html/rfc6455)标准中的WebSocket 协议设计的。

##概述
Conn类型表示一个WebSocket连接。服务器应用程序从HTTP请求处理程序调用Upgrader.Upgrade方法以获取Conn指针：

```golang
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}

func handler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println(err)
        return
    }
    ... Use conn to send and receive messages.
}
```

调用连接的`WriteMessage`和`ReadMessage`方法以一个字节片段发送和接收消息。这段代码展示了如何使用这些方法回显消息：

```golang
for {
    messageType, p, err := conn.ReadMessage()
    if err != nil {
        log.Println(err)
        return
    }
    if err := conn.WriteMessage(messageType, p); err != nil {
        log.Println(err)
        return
    }
}
```

在上面的代码片中，p是[]byte 类型，messageType是一个值为websocket.BinaryMessage或websocket.TextMessage的int类型。

应用程序还可以使用`io.WriteCloser`和`io.Reader`接口发送和接收消息。要发送消息，请调用连接`NextWriter`方法以获取`io.WriteCloser`，将消息写入`writer`并在完成后关闭`writer`。要接收消息，请调用连接`NextReader`方法来获取`io.Reader`并读取，直到返回io.EOF。这段代码展示了如何使用`NextWriter`和`NextReader`方法回显消息：

```
for {
    messageType, r, err := conn.NextReader()
    if err != nil {
        return
    }
    w, err := conn.NextWriter(messageType)
    if err != nil {
        return err
    }
    if _, err := io.Copy(w, r); err != nil {
        return err
    }
    if err := w.Close(); err != nil {
        return err
    }
}
```

## 数据消息
WebSocket协议区分文本和二进制数据消息。文本消息被解释为UTF-8编码文本。二进制消息的解释留给应用程序。

该包使用TextMessage和BinaryMessage整数常量来标识两种数据消息类型。 ReadMessage和NextReader方法返回接收到的消息的类型。 WriteMessage和NextWriter方法的messageType参数指定发送消息的类型。

确保文本消息是有效的UTF-8编码文本是应用程序的责任。

## 控制消息
WebSocket协议定义了三种类型的控制消息：close，ping和pong。调用连接的 WriteControl，WriteMessage 或 NextWriter 方法向对方发送控制消息。
 
连接处理通过调用setCloseHandler方法处理接收到的关闭消息,并且从NextReader,ReadMessage或者读消息的方法中返回一个 *CloseError 指针。关闭处理器默认发送一个关闭消息给对方.

连接通过调用SetPingHandler方法处理接收到的ping消息.默认的ping处理程序向对方发送pong消息。

连接通过SetPongHandler方法处理接收到的pong消息. 默认pong处理器不做任何反应. 如果一个应用发送ping消息, 然后这个应用应该设置一个pong处理器去接收对应的pong消息.

控制消息处理函数从NextReader，ReadMessage和消息阅读器Read方法中调用。当处理程序写入连接时，默认关闭和ping处理程序可以在短时间内阻止这些方法。

应用必须读取连接来处理对方发来的关闭,ping和pong消息. 如果应用程序对其他消息不感兴趣，则应用程序应该启动一个go协程来读取并丢弃来自对等消息的消息。一个简单的示例如下:

```
func readLoop(c *websocket.Conn) {
    for {
        if _, _, err := c.NextReader(); err != nil {
            c.Close()
            break
        }
    }
}
```
## 并发
连接支持一个并发读取器和一个并发的写入器.

应用程序负责确保不超过一个go协程同时调用写入方法（NextWriter，SetWriteDeadline，WriteMessage，WriteJSON，EnableWriteCompression，SetCompressionLevel），并且不超过一个go协程调用读取方法（NextReader，SetReadDeadline，ReadMessage，ReadJSON，SetPongHandler ，SetPingHandler）。

Close和WriteControl方法可以与所有其他方法同时调用。

## Origin注意事项
网络浏览器允许JS应用打开websocket连接到任何主机. 服务器使用浏览器发送的Origin请求头实施一个源策略。

Upgrader 通过调用在CheckOrigin字段中指定函数来检查origin, 一旦CheckOrigin函数返回false, Upgrader方法会使握手失败,并且返回http403的状态码. 

如果CheckOrigin 字段是空, 那么Upgrader 使用一个安全的默认值: 如果存在Origin请求标头并且Origin主机不等于Host请求标头，则握手失败。

废弃的包级upgrade函数不会执行origin的检查,在调用upgrade函数之前，应用程序负责检查Origin标头。

## 压缩试验
每条消息压缩扩展[RFC 7692](https://tools.ietf.org/html/rfc7692)都由该软件包以有限的容量实验性地支持。在Dialer 或者 Upgrader中设置Dialer or Upgrader为true的时候会尝试协商每个消息的压缩支持.

```
var upgrader = websocket.Upgrader{
    EnableCompression: true,
}
```

如果与对方的压缩协商沟通成功,任何消息都会以压缩的形式接收后自动解压. 所有的读取方法都会返回未经压缩的字节.

通过调用Conn相应的方法可以决定每条消息写入连接时是否被压缩.

```
conn.EnableWriteCompression(false)
```

目前这个软件包不支持“上下文接管”压缩。 这意味着消息必须被隔离压缩和解压缩，而不必在消息中保留滑动窗口或字典状态。 有关更多详细信息，请参阅[RFC 7692](https://tools.ietf.org/html/rfc7692)。

