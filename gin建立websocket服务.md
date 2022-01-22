# gin建立websocket服务 #

结合项目介绍一下gin和gorilla结合创建websocket

应用场景：服务端产生消息通知，需要实时推送到所有客户端或者特定的客户端

至于websocket原理这里不在赘述。本文用到一下第三方库

	go get -u github.com/gorilla/websocket
	go get -u github.com/satori/go.uuid

## 服务端 ##

```
// Package ws is to define a websocket server and client connect.
// Author: Arthur Zhang
// Create Date: 20190101
package ws

import (
    "encoding/json"

    "github.com/gorilla/websocket"
)

// ClientManager is a websocket manager
type ClientManager struct {
    Clients    map[*Client]bool
    Broadcast  chan []byte
    Register   chan *Client
    Unregister chan *Client
}

// Client is a websocket client
type Client struct {
    ID     string
    Socket *websocket.Conn
    Send   chan []byte
}

// Message is an object for websocket message which is mapped to json type
type Message struct {
    Sender    string `json:"sender,omitempty"`
    Recipient string `json:"recipient,omitempty"`
    Content   string `json:"content,omitempty"`
}

// Manager define a ws server manager
var Manager = ClientManager{
    Broadcast:  make(chan []byte),
    Register:   make(chan *Client),
    Unregister: make(chan *Client),
    Clients:    make(map[*Client]bool),
}

// Start is to start a ws server
func (manager *ClientManager) Start() {
    for {
        select {
        case conn := <-manager.Register:
            manager.Clients[conn] = true
            jsonMessage, _ := json.Marshal(&Message{Content: "/A new socket has connected."})
            manager.Send(jsonMessage, conn)
        case conn := <-manager.Unregister:
            if _, ok := manager.Clients[conn]; ok {
                close(conn.Send)
                delete(manager.Clients, conn)
                jsonMessage, _ := json.Marshal(&Message{Content: "/A socket has disconnected."})
                manager.Send(jsonMessage, conn)
            }
        case message := <-manager.Broadcast:
            for conn := range manager.Clients {
                select {
                case conn.Send <- message:
                default:
                    close(conn.Send)
                    delete(manager.Clients, conn)
                }
            }
        }
    }
}

// Send is to send ws message to ws client
func (manager *ClientManager) Send(message []byte, ignore *Client) {
    for conn := range manager.Clients {
        if conn != ignore {
            conn.Send <- message
        }
    }
}

func (c *Client) Read() {
    defer func() {
        Manager.Unregister <- c
        c.Socket.Close()
    }()

    for {
        _, message, err := c.Socket.ReadMessage()
        if err != nil {
            Manager.Unregister <- c
            c.Socket.Close()
            break
        }
        jsonMessage, _ := json.Marshal(&Message{Sender: c.ID, Content: string(message)})
        Manager.Broadcast <- jsonMessage
    }
}

func (c *Client) Write() {
    defer func() {
        c.Socket.Close()
    }()

    for {
        select {
        case message, ok := <-c.Send:
            if !ok {
                c.Socket.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            c.Socket.WriteMessage(websocket.TextMessage, message)
        }
    }
}
```

其中


	Start():启动websocket服务
	Send():向连接websocket的管道chan写入数据
	Read():读取在websocket管道中的数据
	Write():通过websocket协议向连接到ws的客户端发送数据

另外需要建立websocket的请求，对于gin我们需要将普通的请求升级为websocket协议

```
// WsPage is a websocket handler
func WsPage(c *gin.Context) {
    // change the reqest to websocket model
    conn, error := (&websocket.Upgrader{CheckOrigin: func(r *http.Request) bool { return true }}).Upgrade(c.Writer, c.Request, nil)
    if error != nil {
        http.NotFound(c.Writer, c.Request)
        return
    }
    // websocket connect
    client := &ws.Client{Id: uuid.NewV4().String(), Socket: conn, Send: make(chan []byte)}

    ws.Manager.Register <- client

    go client.Read()
    go client.Write()
}
```

然后定义路由r.GET("/ws", WsPage).

利用协程的方式来在项目启动时调用Start()就可以建立起websocket的服务端。

启动以后，后端你可以用一下脚本进行测试：

```
package main

import (
    "flag"
    "fmt"
    "net/url"
    "time"

    "github.com/gorilla/websocket"
)

var addr = flag.String("addr", "39.108.105.51:8000", "http service address")

func main() {
    u := url.URL{Scheme: "ws", Host: *addr, Path: "/ws"}
    var dialer *websocket.Dialer

    conn, _, err := dialer.Dial(u.String(), nil)
    if err != nil {
        fmt.Println(err)
        return
    }

    go timeWriter(conn)

    for {
        _, message, err := conn.ReadMessage()
        if err != nil {
            fmt.Println("read:", err)
            return
        }

        fmt.Printf("received: %s\n", message)
    }
}

func timeWriter(conn *websocket.Conn) {
    for {
        time.Sleep(time.Second * 2)
        conn.WriteMessage(websocket.TextMessage, []byte(time.Now().Format("2006-01-02 15:04:05")))
    }
}
```

修改其中的websocket地址即可，前端用onopen建立ws连接即可。

作者：arthur25

链接：https://www.jianshu.com/p/f058fdbdea58

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。