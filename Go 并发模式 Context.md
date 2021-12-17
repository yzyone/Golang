# Go 并发模式 Context #

【导读】本文由来自 Google 研发工程师总结，详细介绍了利用 Context 控制 Go 并发的实践。

在 Go 服务中，每个传入的请求在单独的 goroutine 中处理。请求回调函数通常启动额外的 goroutine 以访问后端，如数据库和 RPC 服务。处理同一请求的一系列 goroutine 通常需要访问请求相关的值，例如端用户的标识、授权令牌和请求截止时间。当请求被取消或超时，处理该请求的所有 goroutine 都应该快速退出，以便系统可以回收它们正在使用的资源。

在 Google，我们开发了一个上下文包，可以轻松地跨越 API 边界，将请求作用域内的值、取消信号和截止时间传递给所有处理请求的 goroutine。该包的公共可用版本为 context。本文描述了如何使用这个包，并提供了一个完整的示例。

## Context ##

context 包的核心是 Context 类型：

```
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

（此描述是精简的；godoc 是权威的。）

Done 方法返回一个 channel，充当传递给 Context 下运行的函数的取消信号：当 channel 关闭时，函数应该放弃它们的工作并返回。Err 方法返回一个错误，表明取消 context 的原因。文章 Pipelines and Cancelation 更详细地讨论了 Done channel 的习惯用法。

Context 没有 Cancel 方法，原因与 Done channel 是只读的一样：接收取消信号的函数通常不是发送信号的函数。特别是当父操作为子操作启动 goroutine 时，子操作不应该有能力取消父操作。相反，WithCancel 函数（如下所述）提供了一种取消新 Context 值的方法。

多个 goroutine 同时使用同一 Context 是安全的。代码可以将单个 Context 传递给任意数量的 goroutine，并取消该 Context 以向所有 goroutine 发送信号。

Deadline 方法允许函数决定是否应该开始工作；如果剩下的时间太少，则可能不值得。代码还可以使用截止时间来设置 I/O 操作超时。

Value 允许 Context 携带请求作用域的数据。为使多个 goroutine 同时使用，这些数据必须是安全的。

## Derived contexts ##

context 包提供了从现有 Context 值派生新 Context 值的函数。这些值形成一个树：当 Context 被取消时，从它派生的所有 Context 也被取消。

Background 是所有 Context 树的根；它永远不会被取消：

```
// Background returns an empty Context. It is never canceled, has no deadline,
// and has no values. Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

WithCancel 和 WithTimeout 返回派生 Context 值，可以比父 Context 更早取消。当请求回调函数返回时，通常会取消与传入请求关联的 Context。WithCancel 还可用于使用多个副本时取消冗余的请求。WithTimeout 用于设置对后端服务器请求的截止时间：

```
// WithCancel returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed or cancel is called.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

WithValue 提供了一种将请求作用域的值与 Context 关联的方法：

```
// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context
```

通过一个示例了解使用 context 包的最佳方法。

## 示例：Google Web 搜索 ##

我们的示例是一个 HTTP 服务器，它处理 URL，如 /search?q=golang&timeout=1s_，将查询 “golang” 转发到 _google web search api 并渲染结果。timeout 参数告诉服务器在该延时后取消请求。

代码分为三个包：

- server 提供了 /search 的 main 函数和请求回调函数。
- userip 提供了从请求中提取用户 IP 地址并将其与 Context 关联的函数。
- google 提供了向 Google 发送查询的 search 函数。

**server 程序**

server 程序处理诸如 /search?q=golang 的请求为 golang 提供 Google 搜索结果。它注册 handlesearch 来处理 /search endpoint_。回调函数创建一个名为 _ctx 的初始 Context，并安排了在回调函数返回时取消它。如果请求包含 timeout URL 参数，则在超时结束时 Context 将自动取消：

```
func handleSearch(w http.ResponseWriter, req *http.Request) {
    // ctx is the Context for this handler. Calling cancel closes the
    // ctx.Done channel, which is the cancellation signal for requests
    // started by this handler.
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    timeout, err := time.ParseDuration(req.FormValue("timeout"))
    if err == nil {
        // The request has a timeout, so create a context that is
        // canceled automatically when the timeout expires.
        ctx, cancel = context.WithTimeout(context.Background(), timeout)
    } else {
        ctx, cancel = context.WithCancel(context.Background())
    }
    defer cancel() // Cancel ctx as soon as handleSearch returns.
```

回调函数从请求中提取查询，并通过调用 userip 包提取客户机的 IP 地址。后端请求需要客户端的 IP 地址，因此 handlesearch 将其附加到 ctx：

```
// Check the search query.
   query := req.FormValue("q")
   if query == "" {
       http.Error(w, "no query", http.StatusBadRequest)
       return
   }

   // Store the user IP in ctx for use by code in other packages.
   userIP, err := userip.FromRequest(req)
   if err != nil {
       http.Error(w, err.Error(), http.StatusBadRequest)
       return
   }
   ctx = userip.NewContext(ctx, userIP)
```

回调函数使用 ctx 和 query 调用 google.Search：

```
// Run the Google search and print the results.
start := time.Now()
results, err := google.Search(ctx, query)
elapsed := time.Since(start)
```

如果搜索成功，回调函数渲染结果：

```
if err := resultsTemplate.Execute(w, struct {
       Results          google.Results
       Timeout, Elapsed time.Duration
   }{
       Results: results,
       Timeout: timeout,
       Elapsed: elapsed,
   }); err != nil {
       log.Print(err)
       return
   }
```

**Package userip**

userip 包提供了从请求中提取用户 IP 地址并将其与 Context 关联的函数。Context 提供键-值映射，其中键和值都是 interface{} 类型。键类型必须支持相等，且值必须安全地供多个 goroutine 同时使用。像 userip 这样的包隐藏映射的细节，并提供对特定 Context 值的强类型访问。

为了避免键冲突， userip 定义了一个未导出的类型 key，并使用此类型的值作为 Context 键：

```
// The key type is unexported to prevent collisions with context keys defined in
// other packages.
type key int

// userIPkey is the context key for the user IP address.  Its value of zero is
// arbitrary.  If this package defined other context keys, they would have
// different integer values.
const userIPKey key = 0
```

fromrequest 从 http.request 中提取 userip 的值：

```
func FromRequest(req *http.Request) (net.IP, error) {
    ip, _, err := net.SplitHostPort(req.RemoteAddr)
    if err != nil {
        return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
}
newcontext 返回一个的携带入参 userip 值的新 Context：

func NewContext(ctx context.Context, userIP net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, userIP)
}
```

fromcontext 从 context 中提取 userip:

```
func FromContext(ctx context.Context) (net.IP, bool) {
    // ctx.Value returns nil if ctx has no value for the key;
    // the net.IP type assertion returns ok=false for nil.
    userIP, ok := ctx.Value(userIPKey).(net.IP)
    return userIP, ok
}
```

**Package google**

该 google.Search 函数向 google web search api 发出 HTTP 请求，并解析 JSON 编码的结果。它接受 Context 参数 ctx_，请求运行时，如果 _ctx.done 关闭，则立刻返回。

google web search api 请求包括搜索 query 和 user ip 作为查询参数：

```
func Search(ctx context.Context, query string) (Results, error) {
    // Prepare the Google Search API request.
    req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
    if err != nil {
        return nil, err
    }
    q := req.URL.Query()
    q.Set("q", query)

    // If ctx is carrying the user IP address, forward it to the server.
    // Google APIs use the user IP to distinguish server-initiated requests
    // from end-user requests.
    if userIP, ok := userip.FromContext(ctx); ok {
        q.Set("userip", userIP.String())
    }
    req.URL.RawQuery = q.Encode()
```

search 使用一个 helper 函数 httpdo 来发出 HTTP 请求；在处理请求或响应时，如果 ctx.done 关闭，将取消调用。search 将闭包传递给 httpdo 处理 HTTP 响应：

```
var results Results
  err = httpDo(ctx, req, func(resp *http.Response, err error) error {
      if err != nil {
          return err
      }
      defer resp.Body.Close()

      // Parse the JSON search result.
      // https://developers.google.com/web-search/docs/#fonje
      var data struct {
          ResponseData struct {
              Results []struct {
                  TitleNoFormatting string
                  URL               string
              }
          }
      }
      if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
          return err
      }
      for _, res := range data.ResponseData.Results {
          results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
      }
      return nil
  })
  // httpDo waits for the closure we provided to return, so it's safe to
  // read results here.
  return results, err
```

httpdo 函数运行 HTTP 请求并在新的 goroutine 中处理其响应。如果 ctx.done 在goroutine 退出之前关闭，将取消请求：

```
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
    // Run the HTTP request in a goroutine and pass the response to f.
    c := make(chan error, 1)
    req = req.WithContext(ctx)
    go func() { c <- f(http.DefaultClient.Do(req)) }()
    select {
    case <-ctx.Done():
        <-c // Wait for f to return.
        return ctx.Err()
    case err := <-c:
        return err
    }
}
```

**根据 Context 调整代码**

许多服务框架提供了包和类型，用于承载请求作用域的值。我们可以定义 Context 接口的新实现，以便在使用现有框架的代码和需要 Context 参数的代码之间架起桥梁。

例如，Gorilla 的 github.com/gorilla/context 包允许处理程序通过提供从 HTTP 请求到键值对的映射，将数据与传入请求相关联。在 gorilla.go，我们提供了一个 Context 实现，其 Value 方法返回与 Gorilla 包中 HTTP 请求相关联的值。

其他包提供了类似于 Context 的取消支持。例如，Tomb 提供了一个 kill 方法，通过关闭一个 dying channel 发出取消信号。tomb 还提供了等待这些 goroutine 退出的方法，类似于 sync.WaitGroup. 在 tomb.go，我们提供了一个 Context 实现，当其父 Context 被取消或提供的 tomb 被杀死时，该 Context 实现被取消。

**总结**

在 Google，我们要求 Go 程序员将 Context 参数作为第一个参数传递给传入和传出请求之间调用路径上的每个函数。这使得许多不同团队开发的 Go 代码能够很好地互操作。它提供了对超时和取消的简单控制，并确保诸如安全凭据之类的关键值正确地传递到程序中。

想要基于 Context 构建的服务器框架应该提供 Context 的实现，以便在其包和那些需要上下文参数的包之间架起桥梁。它们的客户端库接受来自调用代码的 Context。通过为请求作用域的数据和取消建立公共接口，Context 使包开发人员更容易共享代码以创建可伸缩的服务。



转自：

cyningsun.com/01-19-2021/go-concurrency-patterns-context-cn.html