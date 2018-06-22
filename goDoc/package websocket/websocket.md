# package websocket
import "github.com/gorilla/websocket"

Package websocket implements the WebSocket protocol defined in RFC 6455.

Overview
The Conn type represents a WebSocket connection. A server application calls the Upgrader.Upgrade method from an HTTP request handler to get a *Conn:

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
Call the connection's WriteMessage and ReadMessage methods to send and receive messages as a slice of bytes. This snippet of code shows how to echo messages using these methods:

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
In above snippet of code, p is a []byte and messageType is an int with value websocket.BinaryMessage or websocket.TextMessage.

An application can also send and receive messages using the io.WriteCloser and io.Reader interfaces. To send a message, call the connection NextWriter method to get an io.WriteCloser, write the message to the writer and close the writer when done. To receive a message, call the connection NextReader method to get an io.Reader and read until io.EOF is returned. This snippet shows how to echo messages using the NextWriter and NextReader methods:

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
Data Messages
The WebSocket protocol distinguishes between text and binary data messages. Text messages are interpreted as UTF-8 encoded text. The interpretation of binary messages is left to the application.

This package uses the TextMessage and BinaryMessage integer constants to identify the two data message types. The ReadMessage and NextReader methods return the type of the received message. The messageType argument to the WriteMessage and NextWriter methods specifies the type of a sent message.

It is the application's responsibility to ensure that text messages are valid UTF-8 encoded text.

Control Messages
The WebSocket protocol defines three types of control messages: close, ping and pong. Call the connection WriteControl, WriteMessage or NextWriter methods to send a control message to the peer.

Connections handle received close messages by calling the handler function set with the SetCloseHandler method and by returning a *CloseError from the NextReader, ReadMessage or the message Read method. The default close handler sends a close message to the peer.

Connections handle received ping messages by calling the handler function set with the SetPingHandler method. The default ping handler sends a pong message to the peer.

Connections handle received pong messages by calling the handler function set with the SetPongHandler method. The default pong handler does nothing. If an application sends ping messages, then the application should set a pong handler to receive the corresponding pong.

The control message handler functions are called from the NextReader, ReadMessage and message reader Read methods. The default close and ping handlers can block these methods for a short time when the handler writes to the connection.

The application must read the connection to process close, ping and pong messages sent from the peer. If the application is not otherwise interested in messages from the peer, then the application should start a goroutine to read and discard messages from the peer. A simple example is:

func readLoop(c *websocket.Conn) {
    for {
        if _, _, err := c.NextReader(); err != nil {
            c.Close()
            break
        }
    }
}
Concurrency
Connections support one concurrent reader and one concurrent writer.

Applications are responsible for ensuring that no more than one goroutine calls the write methods (NextWriter, SetWriteDeadline, WriteMessage, WriteJSON, EnableWriteCompression, SetCompressionLevel) concurrently and that no more than one goroutine calls the read methods (NextReader, SetReadDeadline, ReadMessage, ReadJSON, SetPongHandler, SetPingHandler) concurrently.

The Close and WriteControl methods can be called concurrently with all other methods.

Origin Considerations
Web browsers allow Javascript applications to open a WebSocket connection to any host. It's up to the server to enforce an origin policy using the Origin request header sent by the browser.

The Upgrader calls the function specified in the CheckOrigin field to check the origin. If the CheckOrigin function returns false, then the Upgrade method fails the WebSocket handshake with HTTP status 403.

If the CheckOrigin field is nil, then the Upgrader uses a safe default: fail the handshake if the Origin request header is present and the Origin host is not equal to the Host request header.

The deprecated package-level Upgrade function does not perform origin checking. The application is responsible for checking the Origin header before calling the Upgrade function.

Compression EXPERIMENTAL
Per message compression extensions (RFC 7692) are experimentally supported by this package in a limited capacity. Setting the EnableCompression option to true in Dialer or Upgrader will attempt to negotiate per message deflate support.

var upgrader = websocket.Upgrader{
    EnableCompression: true,
}
If compression was successfully negotiated with the connection's peer, any message received in compressed form will be automatically decompressed. All Read methods will return uncompressed bytes.

Per message compression of messages written to a connection can be enabled or disabled by calling the corresponding Conn method:

conn.EnableWriteCompression(false)
Currently this package does not support compression with "context takeover". This means that messages must be compressed and decompressed in isolation, without retaining sliding window or dictionary state across messages. For more details refer to RFC 7692.

Use of compression is experimental and may result in decreased performance.

Index
Constants
Variables
func FormatCloseMessage(closeCode int, text string) []byte
func IsCloseError(err error, codes ...int) bool
func IsUnexpectedCloseError(err error, expectedCodes ...int) bool
func IsWebSocketUpgrade(r *http.Request) bool
func ReadJSON(c *Conn, v interface{}) error
func Subprotocols(r *http.Request) []string
func WriteJSON(c *Conn, v interface{}) error
type CloseError
func (e *CloseError) Error() string
type Conn
func NewClient(netConn net.Conn, u *url.URL, requestHeader http.Header, readBufSize, writeBufSize int) (c *Conn, response *http.Response, err error)
func Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header, readBufSize, writeBufSize int) (*Conn, error)
func (c *Conn) Close() error
func (c *Conn) CloseHandler() func(code int, text string) error
func (c *Conn) EnableWriteCompression(enable bool)
func (c *Conn) LocalAddr() net.Addr
func (c *Conn) NextReader() (messageType int, r io.Reader, err error)
func (c *Conn) NextWriter(messageType int) (io.WriteCloser, error)
func (c *Conn) PingHandler() func(appData string) error
func (c *Conn) PongHandler() func(appData string) error
func (c *Conn) ReadJSON(v interface{}) error
func (c *Conn) ReadMessage() (messageType int, p []byte, err error)
func (c *Conn) RemoteAddr() net.Addr
func (c *Conn) SetCloseHandler(h func(code int, text string) error)
func (c *Conn) SetCompressionLevel(level int) error
func (c *Conn) SetPingHandler(h func(appData string) error)
func (c *Conn) SetPongHandler(h func(appData string) error)
func (c *Conn) SetReadDeadline(t time.Time) error
func (c *Conn) SetReadLimit(limit int64)
func (c *Conn) SetWriteDeadline(t time.Time) error
func (c *Conn) Subprotocol() string
func (c *Conn) UnderlyingConn() net.Conn
func (c *Conn) WriteControl(messageType int, data []byte, deadline time.Time) error
func (c *Conn) WriteJSON(v interface{}) error
func (c *Conn) WriteMessage(messageType int, data []byte) error
func (c *Conn) WritePreparedMessage(pm *PreparedMessage) error
type Dialer
func (d *Dialer) Dial(urlStr string, requestHeader http.Header) (*Conn, *http.Response, error)
type HandshakeError
func (e HandshakeError) Error() string
type PreparedMessage
func NewPreparedMessage(messageType int, data []byte) (*PreparedMessage, error)
type Upgrader
func (u *Upgrader) Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header) (*Conn, error)
Examples
IsUnexpectedCloseError
Package Files
client.go client_clone.go compression.go conn.go conn_read.go conn_write.go doc.go json.go mask.go prepared.go proxy.go server.go util.go x_net_proxy.go

Constants
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
Close codes defined in RFC 6455, section 11.7.

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
The message types are defined in RFC 6455, section 11.8.

Variables
var DefaultDialer = &Dialer{
    Proxy:            http.ProxyFromEnvironment,
    HandshakeTimeout: 45 * time.Second,
}
DefaultDialer is a dialer with all fields set to the default values.

var ErrBadHandshake = errors.New("websocket: bad handshake")
ErrBadHandshake is returned when the server response to opening handshake is invalid.

var ErrCloseSent = errors.New("websocket: close sent")
ErrCloseSent is returned when the application writes a message to the connection after sending a close message.

var ErrReadLimit = errors.New("websocket: read limit exceeded")
ErrReadLimit is returned when reading a message that is larger than the read limit set for the connection.

func FormatCloseMessage
func FormatCloseMessage(closeCode int, text string) []byte
FormatCloseMessage formats closeCode and text as a WebSocket close message. An empty message is returned for code CloseNoStatusReceived.

func IsCloseError
func IsCloseError(err error, codes ...int) bool
IsCloseError returns boolean indicating whether the error is a *CloseError with one of the specified codes.

func IsUnexpectedCloseError
func IsUnexpectedCloseError(err error, expectedCodes ...int) bool
IsUnexpectedCloseError returns boolean indicating whether the error is a *CloseError with a code not in the list of expected codes.

Example
func IsWebSocketUpgrade
func IsWebSocketUpgrade(r *http.Request) bool
IsWebSocketUpgrade returns true if the client requested upgrade to the WebSocket protocol.

func ReadJSON
func ReadJSON(c *Conn, v interface{}) error
ReadJSON reads the next JSON-encoded message from the connection and stores it in the value pointed to by v.

Deprecated: Use c.ReadJSON instead.

func Subprotocols
func Subprotocols(r *http.Request) []string
Subprotocols returns the subprotocols requested by the client in the Sec-Websocket-Protocol header.

func WriteJSON
func WriteJSON(c *Conn, v interface{}) error
WriteJSON writes the JSON encoding of v as a message.

Deprecated: Use c.WriteJSON instead.

type CloseError
type CloseError struct {
    // Code is defined in RFC 6455, section 11.7.
    Code int

    // Text is the optional text payload.
    Text string
}
CloseError represents a close message.

func (*CloseError) Error
func (e *CloseError) Error() string
type Conn
type Conn struct {
    // contains filtered or unexported fields
}
The Conn type represents a WebSocket connection.

func NewClient
func NewClient(netConn net.Conn, u *url.URL, requestHeader http.Header, readBufSize, writeBufSize int) (c *Conn, response *http.Response, err error)
NewClient creates a new client connection using the given net connection. The URL u specifies the host and request URI. Use requestHeader to specify the origin (Origin), subprotocols (Sec-WebSocket-Protocol) and cookies (Cookie). Use the response.Header to get the selected subprotocol (Sec-WebSocket-Protocol) and cookies (Set-Cookie).

If the WebSocket handshake fails, ErrBadHandshake is returned along with a non-nil *http.Response so that callers can handle redirects, authentication, etc.

Deprecated: Use Dialer instead.

func Upgrade
func Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header, readBufSize, writeBufSize int) (*Conn, error)
Upgrade upgrades the HTTP server connection to the WebSocket protocol.

Deprecated: Use websocket.Upgrader instead.

Upgrade does not perform origin checking. The application is responsible for checking the Origin header before calling Upgrade. An example implementation of the same origin policy check is:

if req.Header.Get("Origin") != "http://"+req.Host {
	http.Error(w, "Origin not allowed", http.StatusForbidden)
	return
}
If the endpoint supports subprotocols, then the application is responsible for negotiating the protocol used on the connection. Use the Subprotocols() function to get the subprotocols requested by the client. Use the Sec-Websocket-Protocol response header to specify the subprotocol selected by the application.

The responseHeader is included in the response to the client's upgrade request. Use the responseHeader to specify cookies (Set-Cookie) and the negotiated subprotocol (Sec-Websocket-Protocol).

The connection buffers IO to the underlying network connection. The readBufSize and writeBufSize parameters specify the size of the buffers to use. Messages can be larger than the buffers.

If the request is not a valid WebSocket handshake, then Upgrade returns an error of type HandshakeError. Applications should handle this error by replying to the client with an HTTP error response.

func (*Conn) Close
func (c *Conn) Close() error
Close closes the underlying network connection without sending or waiting for a close message.

func (*Conn) CloseHandler
func (c *Conn) CloseHandler() func(code int, text string) error
CloseHandler returns the current close handler

func (*Conn) EnableWriteCompression
func (c *Conn) EnableWriteCompression(enable bool)
EnableWriteCompression enables and disables write compression of subsequent text and binary messages. This function is a noop if compression was not negotiated with the peer.

func (*Conn) LocalAddr
func (c *Conn) LocalAddr() net.Addr
LocalAddr returns the local network address.

func (*Conn) NextReader
func (c *Conn) NextReader() (messageType int, r io.Reader, err error)
NextReader returns the next data message received from the peer. The returned messageType is either TextMessage or BinaryMessage.

There can be at most one open reader on a connection. NextReader discards the previous message if the application has not already consumed it.

Applications must break out of the application's read loop when this method returns a non-nil error value. Errors returned from this method are permanent. Once this method returns a non-nil error, all subsequent calls to this method return the same error.

func (*Conn) NextWriter
func (c *Conn) NextWriter(messageType int) (io.WriteCloser, error)
NextWriter returns a writer for the next message to send. The writer's Close method flushes the complete message to the network.

There can be at most one open writer on a connection. NextWriter closes the previous writer if the application has not already done so.

All message types (TextMessage, BinaryMessage, CloseMessage, PingMessage and PongMessage) are supported.

func (*Conn) PingHandler
func (c *Conn) PingHandler() func(appData string) error
PingHandler returns the current ping handler

func (*Conn) PongHandler
func (c *Conn) PongHandler() func(appData string) error
PongHandler returns the current pong handler

func (*Conn) ReadJSON
func (c *Conn) ReadJSON(v interface{}) error
ReadJSON reads the next JSON-encoded message from the connection and stores it in the value pointed to by v.

See the documentation for the encoding/json Unmarshal function for details about the conversion of JSON to a Go value.

func (*Conn) ReadMessage
func (c *Conn) ReadMessage() (messageType int, p []byte, err error)
ReadMessage is a helper method for getting a reader using NextReader and reading from that reader to a buffer.

func (*Conn) RemoteAddr
func (c *Conn) RemoteAddr() net.Addr
RemoteAddr returns the remote network address.

func (*Conn) SetCloseHandler
func (c *Conn) SetCloseHandler(h func(code int, text string) error)
SetCloseHandler sets the handler for close messages received from the peer. The code argument to h is the received close code or CloseNoStatusReceived if the close message is empty. The default close handler sends a close message back to the peer.

The handler function is called from the NextReader, ReadMessage and message reader Read methods. The application must read the connection to process close messages as described in the section on Control Messages above.

The connection read methods return a CloseError when a close message is received. Most applications should handle close messages as part of their normal error handling. Applications should only set a close handler when the application must perform some action before sending a close message back to the peer.

func (*Conn) SetCompressionLevel
func (c *Conn) SetCompressionLevel(level int) error
SetCompressionLevel sets the flate compression level for subsequent text and binary messages. This function is a noop if compression was not negotiated with the peer. See the compress/flate package for a description of compression levels.

func (*Conn) SetPingHandler
func (c *Conn) SetPingHandler(h func(appData string) error)
SetPingHandler sets the handler for ping messages received from the peer. The appData argument to h is the PING message application data. The default ping handler sends a pong to the peer.

The handler function is called from the NextReader, ReadMessage and message reader Read methods. The application must read the connection to process ping messages as described in the section on Control Messages above.

func (*Conn) SetPongHandler
func (c *Conn) SetPongHandler(h func(appData string) error)
SetPongHandler sets the handler for pong messages received from the peer. The appData argument to h is the PONG message application data. The default pong handler does nothing.

The handler function is called from the NextReader, ReadMessage and message reader Read methods. The application must read the connection to process pong messages as described in the section on Control Messages above.

func (*Conn) SetReadDeadline
func (c *Conn) SetReadDeadline(t time.Time) error
SetReadDeadline sets the read deadline on the underlying network connection. After a read has timed out, the websocket connection state is corrupt and all future reads will return an error. A zero value for t means reads will not time out.

func (*Conn) SetReadLimit
func (c *Conn) SetReadLimit(limit int64)
SetReadLimit sets the maximum size for a message read from the peer. If a message exceeds the limit, the connection sends a close message to the peer and returns ErrReadLimit to the application.

func (*Conn) SetWriteDeadline
func (c *Conn) SetWriteDeadline(t time.Time) error
SetWriteDeadline sets the write deadline on the underlying network connection. After a write has timed out, the websocket state is corrupt and all future writes will return an error. A zero value for t means writes will not time out.

func (*Conn) Subprotocol
func (c *Conn) Subprotocol() string
Subprotocol returns the negotiated protocol for the connection.

func (*Conn) UnderlyingConn
func (c *Conn) UnderlyingConn() net.Conn
UnderlyingConn returns the internal net.Conn. This can be used to further modifications to connection specific flags.

func (*Conn) WriteControl
func (c *Conn) WriteControl(messageType int, data []byte, deadline time.Time) error
WriteControl writes a control message with the given deadline. The allowed message types are CloseMessage, PingMessage and PongMessage.

func (*Conn) WriteJSON
func (c *Conn) WriteJSON(v interface{}) error
WriteJSON writes the JSON encoding of v as a message.

See the documentation for encoding/json Marshal for details about the conversion of Go values to JSON.

func (*Conn) WriteMessage
func (c *Conn) WriteMessage(messageType int, data []byte) error
WriteMessage is a helper method for getting a writer using NextWriter, writing the message and closing the writer.

func (*Conn) WritePreparedMessage
func (c *Conn) WritePreparedMessage(pm *PreparedMessage) error
WritePreparedMessage writes prepared message into connection.

type Dialer
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
A Dialer contains options for connecting to WebSocket server.

func (*Dialer) Dial
func (d *Dialer) Dial(urlStr string, requestHeader http.Header) (*Conn, *http.Response, error)
Dial creates a new client connection. Use requestHeader to specify the origin (Origin), subprotocols (Sec-WebSocket-Protocol) and cookies (Cookie). Use the response.Header to get the selected subprotocol (Sec-WebSocket-Protocol) and cookies (Set-Cookie).

If the WebSocket handshake fails, ErrBadHandshake is returned along with a non-nil *http.Response so that callers can handle redirects, authentication, etcetera. The response body may not contain the entire response and does not need to be closed by the application.

type HandshakeError
type HandshakeError struct {
    // contains filtered or unexported fields
}
HandshakeError describes an error with the handshake from the peer.

func (HandshakeError) Error
func (e HandshakeError) Error() string
type PreparedMessage
type PreparedMessage struct {
    // contains filtered or unexported fields
}
PreparedMessage caches on the wire representations of a message payload. Use PreparedMessage to efficiently send a message payload to multiple connections. PreparedMessage is especially useful when compression is used because the CPU and memory expensive compression operation can be executed once for a given set of compression options.

func NewPreparedMessage
func NewPreparedMessage(messageType int, data []byte) (*PreparedMessage, error)
NewPreparedMessage returns an initialized PreparedMessage. You can then send it to connection using WritePreparedMessage method. Valid wire representation will be calculated lazily only once for a set of current connection options.

type Upgrader
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
Upgrader specifies parameters for upgrading an HTTP connection to a WebSocket connection.

func (*Upgrader) Upgrade
func (u *Upgrader) Upgrade(w http.ResponseWriter, r *http.Request, responseHeader http.Header) (*Conn, error)
Upgrade upgrades the HTTP server connection to the WebSocket protocol.

The responseHeader is included in the response to the client's upgrade request. Use the responseHeader to specify cookies (Set-Cookie) and the application negotiated subprotocol (Sec-WebSocket-Protocol).

If the upgrade fails, then Upgrade replies to the client with an HTTP error response.

Directories
Path	Synopsis
examples/autobahn	Command server is a test server for the Autobahn WebSockets Test Suite.
examples/chat	
examples/command	
examples/filewatch	
Package websocket imports 23 packages (graph) and is imported by 3544 packages. Updated 4 days ago. Refresh now. Tools for package owners.