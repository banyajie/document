

###### 一：从运行服务看底层原理

```go
func main() {
	router := gin.Default()

	router.GET("/test", func(context *gin.Context) {
		context.Writer.Write([]byte("hello"))
	})

  // 服务如何启动。启动后http 请求是如何处理的？
	router.Run("9999")
}

func (engine *Engine) Run(addr ...string) (err error) {
	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine.Handler())
	return
}


// 本质上调用 http 包的 ListenAndServer 方法，所以gin的本质是一个路由处理器，内部定义了路由处理的各种规则 
```



二：请求的处理过程

```go

// http包
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// gin Engine 实现的ServerHttp方法
// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  // 创建上下文对象
	c := engine.pool.Get().(*Context)
  // 初始化上下文对象
	c.writermem.reset(w)
	c.Request = req
	c.reset()
	// 处理请求
	engine.handleHTTPRequest(c)
	// 回收上下文对象
	engine.pool.Put(c)
}

// 处理请求
func (engine *Engine) handleHTTPRequest(c *Context) {
	httpMethod := c.Request.Method
	rPath := c.Request.URL.Path
	unescape := false
	if engine.UseRawPath && len(c.Request.URL.RawPath) > 0 {
		rPath = c.Request.URL.RawPath
		unescape = engine.UnescapePathValues
	}

	if engine.RemoveExtraSlash {
		rPath = cleanPath(rPath)
	}

	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
    // 找到路由
		value := root.getValue(rPath, c.params, c.skippedNodes, unescape)
		if value.params != nil {
			c.Params = *value.params
		}
		if value.handlers != nil {      // 处理器
			c.handlers = value.handlers
			c.fullPath = value.fullPath
      // 调用第一个处理器处理请求。。。。。。。
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
		if httpMethod != http.MethodConnect && rPath != "/" {
			if value.tsr && engine.RedirectTrailingSlash {
				redirectTrailingSlash(c)
				return
			}
			if engine.RedirectFixedPath && redirectFixedPath(c, root, engine.RedirectFixedPath) {
				return
			}
		}
		break
	}

	if engine.HandleMethodNotAllowed {
		for _, tree := range engine.trees {
			if tree.method == httpMethod {
				continue
			}
			if value := tree.root.getValue(rPath, nil, c.skippedNodes, unescape); value.handlers != nil {
				c.handlers = engine.allNoMethod
				serveError(c, http.StatusMethodNotAllowed, default405Body)
				return
			}
		}
	}
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}
```

###### 三：中间件

```go
// c.Next 是一个一个的调用处理器处理请求，中间件也是处理器类型。也是通过c.Next工作

func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}

// 终止剩下的处理器工作
func (c *Context) Abort() {
	c.index = abortIndex
}

// 注册中间件
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}



```

四：Engine 对象初始化

```go
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())  // 默认的中间件
	return engine
}



func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{   
			Handlers: nil,
			basePath: "/",
			root:     true,  // 设置该路由器组为根对象
		},
    
		FuncMap:                template.FuncMap{},
    //为 真。如果只有  /foo 路由存在，会将请求 /foo/ 请求重定向到 /foo  get请求响应301，请求响应307
		RedirectTrailingSlash:  true,
    //如果找不到路由。尝试修复路由。例如 /FOO 和
		RedirectFixedPath:      false,
    //
		HandleMethodNotAllowed: false,
    
    //代理相关配置 
		ForwardedByClientIP:    true,
		RemoteIPHeaders:        []string{"X-Forwarded-For", "X-Real-IP"},
		TrustedPlatform:        defaultPlatform,
		UseRawPath:             false,
		RemoveExtraSlash:       false,
		UnescapePathValues:     true,
		MaxMultipartMemory:     defaultMultipartMemory,
    
    // 不同方法对应的前缀树
		trees:                  make(methodTrees, 0, 9),
    
    //  模板渲染器分隔符
		delims:                 render.Delims{Left: "{{", Right: "}}"},
		secureJSONPrefix:       "while(1);",
		trustedProxies:         []string{"0.0.0.0/0"},
		trustedCIDRs:           defaultTrustedCIDRs,
	}
	engine.RouterGroup.engine = engine
  
  // 分配内存池
	engine.pool.New = func() any {
		return engine.allocateContext()
	}
	return engine
}

```

五：创建路由器组

```go
// 路由组本质上只是一个模板，维护了路径的前缀、中间件信息。使用路由组添加信息，用户省去了重复配置相同前缀和中间件的操作
// 本质上和普通的路由添加没有区别

router := gin.Default()
	g1 := router.Group("/v1")
	{
		g1.GET("/test")
	}

func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
    // 新路由器组继承父路由器组的所有处理器
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}

```

六：路由的注册

```go
	router.GET("/test", func(context *gin.Context) {
		context.Writer.Write([]byte("hello"))
	})

func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	// 根据相对路径计算绝对路径
  absolutePath := group.calculateAbsolutePath(relativePath)
  // 合并处理器
	handlers = group.combineHandlers(handlers)
  // 添加路由。前缀树。
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}




```

###### 七：上下文

```go
//gin 的上下文和 go 云生的上下文不是一回事
// 创建、流程控制、错误管理、元数据管理、请求数据、响应渲染、内容协商

// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
  
	writermem responseWriter
  // 请求对象
	Request   *http.Request
  // 响应对象
	Writer    ResponseWriter

  // 路由参数
	Params   Params
  // 中间件数组
	handlers HandlersChain
  // 当前执行的处理器下标
	index    int8
	fullPath string

	engine       *Engine
	params       *Params
	skippedNodes *[]skippedNode

	// This mutex protects Keys map.
	mu sync.RWMutex

	// Keys is a key/value pair exclusively for the context of each request.
  // 每个请求独有的k v存储
 
	Keys map[string]any

	// Errors is a list of errors attached to all the handlers/middlewares who used this context.
	Errors errorMsgs

	// Accepted defines a list of manually accepted formats for content negotiation.
	Accepted []string

	// queryCache caches the query result from c.Request.URL.Query().
	queryCache url.Values

	// formCache caches c.Request.PostForm, which contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	formCache url.Values

	// SameSite allows a server to define a cookie attribute making it impossible for
	// the browser to send this cookie along with cross-site requests.
	sameSite http.SameSite
}


// Copy returns a copy of the current context that can be safely used outside the request's scope.
// This has to be used when the context has to be passed to a goroutine.
func (c *Context) Copy() *Context {
	cp := Context{
		writermem: c.writermem,
		Request:   c.Request,
		Params:    c.Params,
		engine:    c.engine,
	}
	cp.writermem.ResponseWriter = nil
	cp.Writer = &cp.writermem
	cp.index = abortIndex
	cp.handlers = nil
	cp.Keys = map[string]any{}
	for k, v := range c.Keys {
		cp.Keys[k] = v
	}
	paramCopy := make([]Param, len(cp.Params))
	copy(paramCopy, cp.Params)
	cp.Params = paramCopy
	return &cp
}


```



###### 八：radix 树

![image-20220519175838057](/Users/banyajie/Library/Application Support/typora-user-images/image-20220519175838057.png)

```go
type methodTree struct {
	method string
	root   *node
}

const (
	root nodeType = iota + 1
	param
	catchAll
)

type node struct {
	path      string   //  节点路径
	indices   string   //  节点关键字。分割的分支的第一个字符。
	wildChild bool		 //  是否是独立节点
	nType     nodeType //  类型：
	priority  uint32	 //  节点的优先级、注册的函数数量
	children  []*node // 子节点、child nodes, at most 1 :param style node at the end of the array
	handlers  HandlersChain  // 处理函数
	fullPath  string   // 路由
}

```



##### 总结

- 上下文对象的创建，用到了sync.Pool来复用内存
- gin底层还是用的net/http包。本质是一个路由处理器
- 每个请求方法对应一个radix树
- 添加中间件的过程是切片的追加。所以中间件的执行顺序和添加顺序有关
- 可以为不同的路由组添加不同的中间件
- 路由组本质上只是一个模板。维护了路径前缀、中间件信息
- 新路由组继承了父路由组的所有处理器
- 如果上下文需要被并发使用。需要copy副本