
[原文地址点此](https://godoc.org/github.com/gorilla/websocket#hdr-Overview)
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

压缩是实验性的功能, 可能会导致性能问题.

# 索引
[TOC]
## 例子
## 常量
```
const (
    CloseNormalClosure           = 1000
    CloseGoingAway               = 1001
    CloseProtocolError           = 1002
    CloseUnsupportedData         = 1003
    CloseNoStatusReceived        = 1005
    CloseAbnormalClosure         = 1006
    CloseInvalidFramePayloadData = 1007
    ClosePolicyViolation         = 1008
    CloseMessageTooBig           = 1009
    CloseMandatoryExtension      = 1010
    CloseInternalServerErr       = 1011
    CloseServiceRestart          = 1012
    CloseTryAgainLater           = 1013
    CloseTLSHandshake            = 1015
)
```

关闭码定义在[RFC 6455, 11.7](http://tools.ietf.org/html/rfc6455#section-11.7)中.

```
const (
    // TextMessage denotes a text data message. The text message payload is
    // interpreted as UTF-8 encoded text data.
    TextMessage = 1

    // BinaryMessage denotes a binary data message.
    BinaryMessage = 2

    // CloseMessage denotes a close control message. The optional message
    // payload contains a numeric code and text. Use the FormatCloseMessage
    // function to format a close message payload.
    CloseMessage = 8

    // PingMessage denotes a ping control message. The optional message payload
    // is UTF-8 encoded text.
    PingMessage = 9

    // PongMessage denotes a pong control message. The optional message payload
    // is UTF-8 encoded text.
    PongMessage = 10
)
```

## 变量
```
var DefaultDialer = &Dialer{
    Proxy:            http.ProxyFromEnvironment,
    HandshakeTimeout: 45 * time.Second,
}
```

DefaultDialer 是一个所有字段都会被设为默认值的拨号器.

```
var ErrBadHandshake = errors.New("websocket: bad handshake")
```
当 服务器请求握手无效的时候, ErrBadHandshake会被返回.

```
var ErrCloseSent = errors.New("websocket: close sent")
```
当应用程序在发送了关闭消息之后写入一条消息到连接
会返回ErrReadLimit.

## func [FormatCloseMessage](https://github.com/gorilla/websocket/blob/master/conn.go#L1146)
```
func FormatCloseMessage(closeCode int, text string) []byte
```
FormatCloseMessage 格式化关闭码和文本作为websocket的关闭消息, 如果状态码为CloseNoStatusReceived会返回空消息.

## func [IsCloseError](https://github.com/gorilla/websocket/blob/master/conn.go#L150)

```
func IsCloseError(err error, codes ...int) bool
```
IsCloseError 返回布尔值,表示传入的error是否是*CloseError 指定的出错码.

## func [IsUnexpectedCloseError](https://github.com/gorilla/websocket/blob/master/conn.go#L163)

```
func IsUnexpectedCloseError(err error, expectedCodes ...int) bool
```
IsUnexpectedCloseError 返回布尔值表示error是否在*CloseError中预期的代码中.

## func [IsWebSocketUpgrade](https://github.com/gorilla/websocket/blob/master/server.go#L295)
```golang
func IsWebSocketUpgrade(r *http.Request) bool
```
如果客户端请求升级WebSocket协议,返回true

## func [ReadJSON](https://github.com/gorilla/websocket/blob/master/json.go#L40)
```
func ReadJSON(c *Conn, v interface{}) error
```
ReadJSON 从连接中读取下一个 json编码消息,并将值存入v指向的值中.  
**废弃**: 使用c.ReadJSON代替

## func [Subprotocols](https://github.com/gorilla/websocket/blob/master/server.go#L281)
```
func Subprotocols(r *http.Request) []string
```
Subprotocols返回Sec-Websocket-Protocol头中客户端请求的子协议。

## func [WriteJSON](https://github.com/gorilla/websocket/blob/master/json.go#L15)
```
func WriteJSON(c *Conn, v interface{}) error
```
WriteJSON将v的JSON编码作为消息写入。  
**废弃**: 使用c.WriteJSON代替

## func [CloseError](https://github.com/gorilla/websocket/blob/master/conn.go#L104)
```
type CloseError struct {
    // Code is defined in RFC 6455, section 11.7.
    Code int

    // Text is the optional text payload.
    Text string
}
```
CloseError表示一条关闭消息。
### func (*CloseError) Error
```
func (e *CloseError) Error() string
```

## type [Conn](https://github.com/gorilla/websocket/blob/master/conn.go#L227)
```
type Conn struct {
    // contains filtered or unexported fields
}
```
Conn类型表示一个WebSocket连接。

### func [NewClient](https://github.com/gorilla/websocket/blob/master/client.go#L37)
```
func NewClient(netConn net.Conn, u *url.URL, requestHeader http.Header, readBufSize, writeBufSize int) (c *Conn, response *http.Response, err error)
```
NewClient 用给定的网络练剑创建一个新客户端连击. URL参数u 指定这个主机和请求的URI. 使用请求头去指定origin, 子协议(Sec-WebSocket-Protocol) 和缓存(Cookie). 使用头部返回去获取选定的子协议(Sec-WebSocket-Protocol)和缓存(Set-Cookie).

如果websocket握手失败，ErrBadHandshake将返回一个非nil *http.Response，以便调用者能够处理重定向、身份验证等。  
**废弃**: 使用Dialer代替

### func [Upgrade](https://github.com/gorilla/websocket/blob/master/server.go#L267)

```
func Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header, readBufSize, writeBufSize int) (*Conn, error)
```

Upgrade 升级http服务器链接到websocket协议.
  
**废弃**: 使用websocket.Upgrader代替  

Upgrde 不执行origin检查. 应用程序负责在调用升级之前检查源文件头。同一origin策略检查的一个例子是:

```golang
if req.Header.Get("Origin") != "http://"+req.Host {
	http.Error(w, "Origin not allowed", http.StatusForbidden)
	return
}
```
 
如果端点支持子协议，那么应用程序负责协商连接上使用的协议。使用Subprotocols()函数获取客户端请求的子协议。使用sec-websocket-protocol响应头来指定应用程序所选择的子协议。

responseHeader包含在对客户端升级请求的响应中。使用responseHeader指定cookie (Set-Cookie)和协商子协议(Sec-Websocket-Protocol)。


连接缓冲IO到底层网络连接。readBufSize和writeBufSize参数指定要使用的缓冲区的大小。消息可以大于缓冲区。


如果请求不是有效的WebSocket握手，那么Upgrade将返回一个类型为HandshakeError的错误。应用程序应该通过使用HTTP错误响应来回答客户端来处理这个错误。

### func (*Conn) [Close](https://github.com/gorilla/websocket/blob/master/conn.go#L347)

```
func (c *Conn) Close() error
```
Close关闭底层网络链接,不需要发送或者等待关闭消息.

### func (*Conn) [CloseHandler](https://github.com/gorilla/websocket/blob/master/conn.go#L1044)
```
func (c *Conn) CloseHandler() func(code int, text string) error
```
CloserHandler 返回当当前关闭处理器.


### func(*Conn) [EnableWriteCompression](https://github.com/gorilla/websocket/blob/master/conn.go#L1128)
```
func (c *Conn) EnableWriteCompression(enable bool)
```
EnableWriteCompression 启用和禁用后续文本和二级制消息的写入压缩. 如果没有与对方协商压缩,则此方法为空操作.

### func(*Conn) [LocalAddr](https://github.com/gorilla/websocket/blob/master/conn.go#L352)
LocalAddr 返回本地的IP地址

### func(*Conn) [NextReader](https://github.com/gorilla/websocket/blob/master/conn.go#L928)
```
func (c *Conn) NextReader() (messageType int, r io.Reader, err error)
```
NextReader 返回从对方收到的后续数据消息. 返回的messageType 是 TextMessage 或者BinaryMessage.

在连接上最多只有一个打开的阅读器. 如果应用程序还没销毁它,NextReader会丢弃以前的消息.

当此方法返回非nil错误值时，应用程序必须跳出应用程序的读循环。此方法返回的错误是永久性的。一旦该方法返回一个非nil错误，对该方法的所有后续调用都将返回相同的错误。


### func (*Conn) [NextWriter](https://github.com/gorilla/websocket/blob/master/conn.go#L489)

```golang
func (c *Conn) NextWriter(messageType int) (io.WriteCloser, error)
```
NextWriter返回一个写入器,由于发送后续消息. 写入器的关闭方法会刷新完整的消息到网络.

在连接上最多只有一个打开的写入器. 如果应用程序还没关闭先前的写入器,NextWriter 将会关闭先前的写入器

所有消息类型(TextMessage, BinaryMessage, CloseMessage, PingMessage and PongMessage)都被支持

### func (*Conn) [PingHandler](https://github.com/gorilla/websocket/blob/master/conn.go#L1074)

```
func (c *Conn) PingHandler() func(appData string) error
```
PingHandler 返回当前ping处理器

### func (*Conn) [PongHandler](https://github.com/gorilla/websocket/blob/master/conn.go#L1101)
```
func (c *Conn) PongHandler() func(appData string) error
```
PongHandler 返回当前pong处理器

### func (*Conn) [ReadJSON](https://github.com/gorilla/websocket/blob/master/json.go#L49)
```
func (c *Conn) ReadJSON(v interface{}) error
```
ReadJSON 从连接中读取后续json编码格式的消息. 并将其存储在v指向的值中。  
有关json转换到Go值的详细信息，请参阅编码/json Unmarshal函数的文档。

### func (*Conn) [ReadMessage](https://github.com/gorilla/websocket/blob/master/conn.go#L1018)
```
func (c *Conn) ReadMessage() (messageType int, p []byte, err error)
```
ReadMessage是一个助手方法，用于使用NextReader获取读取器并从该读取器读取到缓冲区。

### func (*Conn) [RemoteAddr](https://github.com/gorilla/websocket/blob/master/conn.go#L357)
```
var (c *Conn) RemoteAddr() net.Addr
```
ReomoteAddr返回远端主机地址.

### func (*Conn) SetCloseHandler
```
func (c *Conn) SetCloseHandler(h func(code int, text string) error)
```
 
SetCloseHandler 设置处理器,用户处理对方发送过来的关闭消息, h参数是接收到的关闭码,或者当关闭消息为空的时候,h的值为CloseNoStatusReceived.

这个处理器函数在 NextReader,ReadMessage和消息阅读器的读取方法里调用. 应用程序必须读取连接来处理上一节中控制消息中描述的关闭消息.

当收到关闭消息时,此连接读取方法返回一个CloseError. 大多数应用应当把关闭消息当做常规错误消息处理的一部分来处理. 当应用必须在向对方返回关闭消息之前执行某些操作, 应用只能设置一个关闭处理器.

### func (*Conn) [SetCompressionLevel](https://github.com/gorilla/websocket/blob/master/conn.go#L1136)
```
func (c *Conn) SetCompressionLevel(level int) error
```
SetCompressionLevel为后续文本和二进制消息设置flate（这里并不知道怎么翻译）压缩级别。 如果未与对方协商压缩，则此函数为noop。 有关压缩级别的说明，请参阅compress / flate包。

### func(*Conn) [SetPingHandler](https://github.com/gorilla/websocket/blob/master/conn.go#L1085)
```
func (c *Conn) SetPingHandler(h func(appData string) error)
```
SetPingHandler设置从对方接收的ping消息的处理程序。 h的appData参数是PING消息应用程序数据。 默认的ping处理程序将pong发送给对方。

处理程序函数从NextReader，ReadMessage和消息读取器读取方法调用。 应用程序必须读取连接以处理ping消息，如上面的控制消息部分所述。

### func(*Conn) [SetPongHandler](https://github.com/gorilla/websocket/blob/master/conn.go#L1112)
```
func (c *Conn) SetPongHandler(h func(appData string) error)
```

SetPongHandler设置从对方接收的pong消息的处理程序。 h的appData参数是PONG消息应用程序数据。 默认的pong处理程序什么都不做。

处理程序函数从NextReader，ReadMessage和消息读取器读取方法调用。 应用程序必须读取处理pong消息的连接，如上面的控制消息部分所述。

### func(*Conn) [SetReadDeadline](https://github.com/gorilla/websocket/blob/master/conn.go#L1032)

SetReadDeadline设置底层网络连接的读取截止日期。 读取超时后，websocket连接状态已损坏，所有将来的读取都将返回错误。 t的零值意味着读取不会超时

### func (*Conn) [SetReadLimit](https://github.com/gorilla/websocket/blob/master/conn.go#L1039)
```
func (c *Conn) SetReadLimit(limit int64)
```
SetReadLimit设置从对方读取的消息的最大限制。 如果消息超出限制，则连接会向对方发送关闭消息，并将ErrReadLimit返回给应用程序。

### func (*Conn) [SetWriteDeadline](https://github.com/gorilla/websocket/blob/master/conn.go#L761)
```
func (c *Conn) SetWriteDeadline(t time.Time) error
```
SetWriteDeadline设置底层网络连接的写入期限。 写入超时后，websocket状态已损坏，所有将来的写入都将返回错误。 t的零值表示写入不会超时。

### func (*Conn) [Subprotocol](https://github.com/gorilla/websocket/blob/master/conn.go#L341)
```
func (c *Conn) Subprotocol() string
```
Subprotocol返回协商的连接协议。

### func (*Conn) [UnderlyingConn](https://github.com/gorilla/websocket/blob/master/conn.go#L1121)
```
func (c *Conn) UnderlyingConn() net.Conn
```
UnderlyingConn返回内部net.Conn。 这可用于进一步修改连接特定标志


### func (*Conn) [WriteControl](https://github.com/gorilla/websocket/blob/master/conn.go#L401)
```
func (c *Conn) WriteControl(messageType int, data []byte, deadline time.Time) error
```
WriteControl在给定的截止日期之前写入控制消息。 允许的消息类型是CloseMessage，PingMessage和PongMessage。

### func (*Conn) [WriteJSON](https://github.com/gorilla/websocket/blob/master/json.go#L23)
```
func (c *Conn) WriteJSON(v interface{}) error
```
WriteJSON将v的JSON编码之后的文本作为消息。

有关将Go值转换为JSON的详细信息，请参阅encoding / json Marshal的文档。

### func (*Conn) [WriteMessage](https://github.com/gorilla/websocket/blob/master/conn.go#L732)
```
func (c *Conn) WriteMessage(messageType int, data []byte) error
```
WriteMessage是一个辅助方法，用于使用NextWriter获取编写器，编写消息并关闭编写器。

### func (*Conn) [WritePreparedMessage](https://github.com/gorilla/websocket/blob/master/conn.go#L709)
```
func (c *Conn) WritePreparedMessage(pm *PreparedMessage) error
```
WritePreparedMessage将准备好的消息写入连接。

## type [Dialer](https://github.com/gorilla/websocket/blob/master/client.go#L49)
```
type Dialer struct {
    // NetDial specifies the dial function for creating TCP connections. If
    // NetDial is nil, net.Dial is used.
    NetDial func(network, addr string) (net.Conn, error)

    // Proxy specifies a function to return a proxy for a given
    // Request. If the function returns a non-nil error, the
    // request is aborted with the provided error.
    // If Proxy is nil or returns a nil *URL, no proxy is used.
    Proxy func(*http.Request) (*url.URL, error)

    // TLSClientConfig specifies the TLS configuration to use with tls.Client.
    // If nil, the default configuration is used.
    TLSClientConfig *tls.Config

    // HandshakeTimeout specifies the duration for the handshake to complete.
    HandshakeTimeout time.Duration

    // ReadBufferSize and WriteBufferSize specify I/O buffer sizes. If a buffer
    // size is zero, then a useful default size is used. The I/O buffer sizes
    // do not limit the size of the messages that can be sent or received.
    ReadBufferSize, WriteBufferSize int

    // Subprotocols specifies the client's requested subprotocols.
    Subprotocols []string

    // EnableCompression specifies if the client should attempt to negotiate
    // per message compression (RFC 7692). Setting this value to true does not
    // guarantee that compression will be supported. Currently only "no context
    // takeover" modes are supported.
    EnableCompression bool

    // Jar specifies the cookie jar.
    // If Jar is nil, cookies are not sent in requests and ignored
    // in responses.
    Jar http.CookieJar
}
```
拨号程序包含用于连接WebSocket服务器的选项。

### func (*Dialer) [Dial](https://github.com/gorilla/websocket/blob/master/client.go#L125)

```
func (d *Dialer) Dial(urlStr string, requestHeader http.Header) (*Conn, *http.Response, error)
```
拨号创建新的客户端连接。 使用requestHeader指定源（Origin），子协议（Sec-WebSocket-Protocol）和cookie（Cookie）。 使用response.Header获取所选的子协议（Sec-WebSocket-Protocol）和cookie（Set-Cookie）。

如果WebSocket握手失败，则返回ErrBadHandshake以及非nil 的*http.Response，以便调用者可以处理重定向，身份验证等。 响应正文可能不包含整个响应，也不需要由应用程序关闭。


## type [HandshakeError](https://github.com/gorilla/websocket/blob/master/server.go#L18)
```
type HandshakeError struct {
    // contains filtered or unexported fields
}
```
HandshakeError描述了来自对等方的握手错误。

### func (HandshakeError) [Error](https://github.com/gorilla/websocket/blob/master/server.go#L22)
```
func (e HandshakeError) Error() string
```
## type [PreparedMessage](https://github.com/gorilla/websocket/blob/master/prepared.go#L19)
```
type PreparedMessage struct {
    // contains filtered or unexported fields
}
```
PreparedMessage缓存在串行的消息负载上(PS.这里翻译可能有问题)。 使用PreparedMessage有效地将消息有效负载发送到多个连接。 PreparedMessage在使用压缩时特别有用，因为对于给定的一组压缩选项，CPU和内存昂贵的压缩操作可以执行一次。

### func [NewPreparedMessage](https://github.com/gorilla/websocket/blob/master/prepared.go#L44)

```
func NewPreparedMessage(messageType int, data []byte) (*PreparedMessage, error)
```
NewPreparedMessage返回初始化的PreparedMessage。你可以使用WritePreparedMessage方法将其发送到连接。 对于一组当前连接选项，有效的线表示将仅延迟计算一次。

## type [Upgrader](https://github.com/gorilla/websocket/blob/master/server.go#L26)
```
type Upgrader struct {
    // HandshakeTimeout specifies the duration for the handshake to complete.
    HandshakeTimeout time.Duration

    // ReadBufferSize and WriteBufferSize specify I/O buffer sizes. If a buffer
    // size is zero, then buffers allocated by the HTTP server are used. The
    // I/O buffer sizes do not limit the size of the messages that can be sent
    // or received.
    ReadBufferSize, WriteBufferSize int

    // Subprotocols specifies the server's supported protocols in order of
    // preference. If this field is set, then the Upgrade method negotiates a
    // subprotocol by selecting the first match in this list with a protocol
    // requested by the client.
    Subprotocols []string

    // Error specifies the function for generating HTTP error responses. If Error
    // is nil, then http.Error is used to generate the HTTP response.
    Error func(w http.ResponseWriter, r *http.Request, status int, reason error)

    // CheckOrigin returns true if the request Origin header is acceptable. If
    // CheckOrigin is nil, then a safe default is used: return false if the
    // Origin request header is present and the origin host is not equal to
    // request Host header.
    //
    // A CheckOrigin function should carefully validate the request origin to
    // prevent cross-site request forgery.
    CheckOrigin func(r *http.Request) bool

    // EnableCompression specify if the server should attempt to negotiate per
    // message compression (RFC 7692). Setting this value to true does not
    // guarantee that compression will be supported. Currently only "no context
    // takeover" modes are supported.
    EnableCompression bool
}
```
Upgrader指定用于将HTTP连接升级到WebSocket连接的参数。

### func (*Upgrader) [Upgrade](https://github.com/gorilla/websocket/blob/master/server.go#L110)

```
func (u *Upgrader) Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header) (*Conn, error)
```
升级会将HTTP服务器连接升级到WebSocket协议。

responseHeader包含在对客户端升级请求的响应中。 使用responseHeader指定cookie（Set-Cookie）和应用程序协商的子协议（Sec-WebSocket-Protocol）。

如果升级失败，则升级将使用HTTP错误响应回复客户端。
