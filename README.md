
## 前言


这里是白泽，我将利用一个系列，为你分享如何基于 websocket 协议的 rfc 文档，编写一个库的过程。并从0开始写一遍 gorilla/websocket 这个库，从中你可以学习到 websocket 库中高质量、高性能的写法（多协程、缓冲池使用）。


仓库地址：[https://github.com/gorilla/websocket，🌟数量：22\.8k](https://github.com)


项目体量不大、核心代码5k，虽然难度较高，但在引导下也可以完成。


![image-20241221234238012](https://baize-blog-images.oss-cn-shanghai.aliyuncs.com/img/image-20241221234238012.png)



> B站：[白泽talk](https://github.com)
> 
> 
> 公众号：白泽talk
> 
> 
> 开源 Golang 学习仓库：[https://github.com/BaiZe1998/go\-learning](https://github.com):[wgetCloud机场](https://tabijibiyori.org)
> 
> 
> 开源短视频应用 DouTok：[https://github.com/cloudzenith/DouTok](https://github.com)


## WebSocket 协议


这是第一课，本文将详细介绍 WebSocket 协议的帧结构、握手过程、数据传输机制及其在实际应用中的优势。并以 Golang 的 websocket 开源库作为案例进行前瞻，探究协议是如何用 Golang 代码实现的，以及如何使用这个库，进行 websocket 通信。


### 一、WebSocket 协议概述


WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。与 HTTP 相比，WebSocket 无需像 HTTP 那样每次通信都需要建立新的连接，从而大大减少了延迟和资源消耗。它使得客户端和服务器之间的数据交换变得更加高效和实时。


### 二、WebSocket 帧结构


作为 Go 语言 🌟 数最多的 websocket 库，本质上是通过 Golang，实现了 websocket rfc 文档定义的消息格式、以及交互过程。


rfc：[https://datatracker.ietf.org/doc/html/rfc6455\#section\-5\.2](https://github.com)


![image-20241221223203717](https://baize-blog-images.oss-cn-shanghai.aliyuncs.com/img/image-20241221223203717.png)


这个库的核心实现包含在几个文件中（client.go/serve.go/conn.go）。其中用于定义 websocket 链接的 conn.go 文件中，定义了一些常量，用于表示各个标志位，乍一眼你可能看不明白。



```
const (
	// Frame header byte 0 bits from Section 5.2 of RFC 6455
	finalBit = 1 << 7
	rsv1Bit  = 1 << 6
	rsv2Bit  = 1 << 5
	rsv3Bit  = 1 << 4

	// Frame header byte 1 bits from Section 5.2 of RFC 6455
	maskBit = 1 << 7

	maxFrameHeaderSize         = 2 + 8 + 4 // Fixed header + length + mask
	maxControlFramePayloadSize = 125

	writeWait = time.Second

	defaultReadBufferSize  = 4096
	defaultWriteBufferSize = 4096

	continuationFrame = 0
	noFrame           = -1
)

```

WebSocket 协议通过帧（Frame）来传输数据。每个帧都包含一系列固定的字段，用于标识帧的类型、长度、掩码以及实际的数据内容。


![image-20241221205623911](https://baize-blog-images.oss-cn-shanghai.aliyuncs.com/img/image-20241221205623911.png)


##### 1\. FIN


FIN 字段是一个 1 位的标志位，用于指示当前帧是否为消息中的最后一个片段。如果消息仅由一个片段组成，该位也应被设置为 1。


##### 2\. RSV1, RSV2, RSV3


RSV1、RSV2 和 RSV3 是三个 1 位的保留位，它们必须为 0，除非在 WebSocket 握手阶段已经协商了具有特定含义的扩展。如果使用了扩展，并且这些位被赋予了特定的意义，那么它们将用于指示该帧是否遵循了这些扩展的特定规则。


##### 3\. Opcode


Opcode 字段是一个 4 位的操作码，用于定义有效载荷数据的含义。不同的操作码代表不同类型的帧：


* %x0：表示连续帧（Continuation Frame），用于将消息分割成多个片段。
* %x1：表示文本帧（Text Frame），包含 UTF\-8 编码的文本数据。
* %x2：表示二进制帧（Binary Frame），包含二进制数据。
* %x8：表示连接关闭帧（Connection Close Frame），用于关闭连接。
* %x9：表示 Ping 帧，用于连接检测。
* %xA：表示 Pong 帧，作为对 Ping 帧的响应。


##### 4\. Mask


Mask 字段是一个 1 位的标志位，用于指示是否对有效载荷数据进行了掩码处理。在客户端发送到服务器的帧中，该位必须设置为 1，并附带一个掩码键（Masking\-key）。服务器发送到客户端的帧则不应被掩码，因此该位应为 0。


##### 5\. Payload length


Payload length 字段用于表示有效载荷数据的总长度。它可以是 7 位、7\+16 位或 7\+64 位，具体取决于数据的长度：


* 如果长度小于或等于 125 字节，则使用 7 位表示。
* 如果长度在 126 到 65,535 字节之间，则前 7 位设置为 126，并使用随后的 16 位来表示长度。
* 如果长度超过 65,535 字节，则前 7 位设置为 127，并使用随后的 64 位来表示长度。


##### 6\. Masking\-key


如果 Mask 位为 1，则 Masking\-key 字段存在，并包含 32 位的掩码。该掩码用于对有效载荷数据进行掩码处理，以确保数据的安全性。


##### 7\. Payload data


Payload data 字段包含实际要传输的数据，它由扩展数据（Extension data）和应用数据（Application data）组成。扩展数据是可选的，其长度和格式取决于在 WebSocket 握手阶段协商的扩展。应用数据则包含了实际要传输的数据，如文本消息、二进制数据等。


🤔 思考一下：



```
提问：websocket/conn.go:31 中，maxFrameHeaderSize = 2 + 8 + 4 的含义是？

回答：2字节的固定长度 + 8字节的最大负载长度(可变) + 4字节掩码（可变）。

```

### 三、WebSocket 握手过程


WebSocket 的握手过程是基于 HTTP 的。当客户端希望与服务器建立 WebSocket 连接时，它会首先发送一个 HTTP 请求到服务器。这个请求包含了几个关键的头部字段，用于指示客户端希望升级到 WebSocket 协议。



```
func main() {
	flag.Parse()
	hub := newHub()
	go hub.run()
	http.HandleFunc("/", serveHome) // HTTP 请求
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWs(hub, w, r) // HTTP 协议升级成 ws 协议
	})
	err := http.ListenAndServe(*addr, nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}

// serveWs handles websocket requests from the peer.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil) // 劫持 conn，并升级为 ws 协议的函数，这部分是 websocket 库实现的
	if err != nil {
		log.Println(err)
		return
	}
	client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
	client.hub.register <- client

	// Allow collection of memory referenced by the caller by doing all work in
	// new goroutines.
	go client.writePump()
	go client.readPump()
}

```

服务器在接收到这个请求后，会验证请求的有效性，并返回一个包含 `Upgrade: websocket` 和 `Connection: Upgrade` 头部的 HTTP 响应。这个响应表示服务器同意升级到 WebSocket 协议，并且连接已经成功建立。


### 四、WebSocket 数据传输机制


一旦 WebSocket 连接建立成功，客户端和服务器就可以通过帧来传输数据了。每个帧都包含上述的帧结构，用于标识帧的类型、长度、掩码以及实际的数据内容。


在数据传输过程中，客户端和服务器可以自由地发送和接收数据帧，实现全双工通信。这种通信方式使得实时应用能够高效地处理数据交换，减少延迟和资源消耗。


### 五、一个聊天室的通信 demo


服务端通过维护一个 hub 管理所有客户端活跃的链接。



```
// 比如服务端通过维护一个 hub 管理所有客户端活跃的链接。
type Hub struct {
	// Registered clients.
	clients map[*Client]bool

	// Inbound messages from the clients.
	broadcast chan []byte

	// Register requests from the clients.
	register chan *Client

	// Unregister requests from clients.
	unregister chan *Client
}

```

服务端启动协程监听客户端实例的注册，以及接收来自客户端的消息，并且广播给所有的客户端。



```
func (h *Hub) run() {
    for {
       select {
       case client := <-h.register: // 升级成 ws 协议之后，注册 client 到 hub
          h.clients[client] = true
       case client := <-h.unregister: // 连接断开则注销 client 实例，释放资源
          if _, ok := h.clients[client]; ok {
             delete(h.clients, client)
             close(client.send)
          }
       case message := <-h.broadcast: // 一个阻塞的 channal，接收来自 client 的需要广播的消息
          for client := range h.clients { // 发送给所有的 client 实例
             select {
             case client.send <- message:
             default:
                close(client.send)
                delete(h.clients, client)
             }
          }
       }
    }
}

```

客户端的结构。



```
// Client is a middleman between the websocket connection and the hub.
type Client struct {
    hub *Hub

    // The websocket connection.
    conn *websocket.Conn

    // Buffered channel of outbound messages.
    send chan []byte
}

```

从 http 协议升级成 ws 协议之后，将 client 注册到 hub 中。



```
http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
    serveWs(hub, w, r) // HTTP 协议升级成 ws 协议
})

// serveWs handles websocket requests from the peer.
func serveWs(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil) // 升级为 ws 协议，劫持 conn，后续进行全双工通信
	if err != nil {
		log.Println(err)
		return
	}
	client := &Client{hub: hub, conn: conn, send: make(chan []byte, 256)}
	client.hub.register <- client

	// Allow collection of memory referenced by the caller by doing all work in
	// new goroutines.
	go client.writePump() // 开启 client 向 web 端回写消息的协程
	go client.readPump() // 开启 client 监听来自 web 端的消息，并发送给 hub，进行广播
}

```

注册在 hub 中的客户端实例，不断监听读取来自 web 端的消息，并转发给 hub。



```
func (c *Client) readPump() {
    defer func() {
       c.hub.unregister <- c
       c.conn.Close()
    }()
    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error { c.conn.SetReadDeadline(time.Now().Add(pongWait)); return nil })
    for {
       _, message, err := c.conn.ReadMessage() // 读取来自 web 端的消息
       if err != nil {
          if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
             log.Printf("error: %v", err)
          }
          break
       }
       message = bytes.TrimSpace(bytes.Replace(message, newline, space, -1)) // 格式化 
       c.hub.broadcast <- message
    }
}

```

注册在 hub 中的客户端实例，同时监听发送给自己的消息，并且写入 conn 句柄，将消息再次转发回 web 端的实例。



```
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
       ticker.Stop()
       c.conn.Close()
    }()
    for {
       select {
       case message, ok := <-c.send: // 是否有需要回写给 web 端的消息
          c.conn.SetWriteDeadline(time.Now().Add(writeWait))
          if !ok {
             // The hub closed the channel.
             c.conn.WriteMessage(websocket.CloseMessage, []byte{})
             return
          }

          w, err := c.conn.NextWriter(websocket.TextMessage)
          if err != nil {
             return
          }
          w.Write(message) // 发送文本消息

          // Add queued chat messages to the current websocket message.
          n := len(c.send)
          for i := 0; i < n; i++ {
             w.Write(newline)
             w.Write(<-c.send)
          }

          if err := w.Close(); err != nil {
             return
          }
       case <-ticker.C: // 周期发送 ping 消息
          c.conn.SetWriteDeadline(time.Now().Add(writeWait))
          if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
             return
          }
       }
    }
}

```

### 六、WebSocket 协议在实际应用中的优势


1. **实时性**：WebSocket 协议允许客户端和服务器之间进行实时的双向通信，使得应用能够更快地响应用户操作。
2. **高效性**：与 HTTP 相比，WebSocket 无需每次通信都建立新的连接，从而大大减少了延迟和资源消耗。
3. **灵活性**：WebSocket 协议支持文本和二进制数据的传输，使得应用能够处理多种类型的数据。
4. **安全性**：WebSocket 协议提供了对数据的掩码处理，以确保数据在传输过程中的安全性。


## 小结


未完待续，期待你的关注。


