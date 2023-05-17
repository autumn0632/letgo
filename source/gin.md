

# 什么是gin

核心组件包括`gin.Engine`，`gin.Context`，`gin.RouterGroup`等


`example`


# gin源码分析

## 框架启动流程

通过以下流程，可以快速启动一个gin服务：
1. 实例化gin框架
2. 注册路由
3. 启动服务

如下所示：
```go
import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Contest) {
        c.JSON(http.StatusOK, gin.H{
            "message":"pong"
        })
    })
    r.Run() // listen and server on 0.0.0.0:8080
}

```

### gin.Engine

`gin.Engine`是gin框架的实例，可以通过`gin.New()`或`gin.Default()`获取。结构体和相关方法主要定义在`gin.go`文件中。

结构体字段主要包括：
* 控制某些功能开关的bool类型值
* 组合RouterGroup结构，实现其功能
* pool缓存池，控制gin.Context分配与释放，提升http请求时性能
* trees路由树，存储路由和handle方法映射

主要方法有：

* func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request): 实现http.Handler接口，从而将 http 请求流入gin框架进行处理
* func (engine *Engine) handleHTTPRequest(c *Context) ：处理请求
* func (engine *Engine) Run(addr ...string) (err error) ：启动服务
* func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes ：添加中间件

```go
    type Engine struct {
        RouterGroup                     // 存储所有中间件，包括请求路径

        //...

        trees methodTrees

        pool sync.Pool

    }

    func New() *Engine {
        engine := &Engine{
            RouterGroup: RouterGroup{
                Handlers: nil,
                basePath: "/",
                root: true,
            },
            // ...
            trees: make(methodTrees, 0, 9),
        }
        engine.RouterGroup.engine = engine
        engine.pool.New = func any {
            return engine.allocateContext(engine.maxParams)
        }
        return engine
    }

    func Default() *Engine {
	    debugPrintWARNINGDefault()
	    engine := New()
	    engine.Use(Logger(), Recovery())
	    return engine
    }

    // ServeHTTP conforms to the http.Handler interface.
    func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	    c := engine.pool.Get().(*Context)
	    c.writermem.reset(w)
	    c.Request = req
	    c.reset()

	    engine.handleHTTPRequest(c)

	    engine.pool.Put(c)
    }


```

## http请求处理流程

### gin.Context

`gin.Context`主要实现了对`Request`和`Response`的封装以及一些参数的传递。gin框架初始化的过程内，会初始化Context缓存池，即`engine.pool`，用来优化http请求时的性能。相关数据结构和方法主要定义在`context.go`文件中。

主要方法包括：
1. 处理HTTP的Request请求数据
    * func (c *Context) Param(key string) string：获取url参数
    * 获取`Get`请求相关数据（querystring中的参数）
    * 获取`POST`请求相关数据（form表单数据）
    * 获取上传文件
    * 获取Cookie数据
    * Bind数据，ShouldBind,MustBind

2. 返回渲染响应数据：`json`/`String`
    * func (c *Context) JSON(code int, obj interface{})

3. 流程控制

    Gin中保存了路由处理信息，在流程控制中统一来执行所保存的信息。在实现的过程中主要调用了`Next`和`Abort`两个方法

4. 元数据管理

    专门用于为此上下文存储新的键值对。存储在Context中的Keys数据字段中

```go
    type Context struct {
        Request *http.Request       // http请求
        Writer  ResponseWriter      // http响应
        
        Params  Params              // URL参数，有key/value组成
        
        handlers HandlersChain      // HandleFunc数组，责任链模式队请求进行处理
        index    int8               // HandleFunc数组的索引
        
        engine  *Engine             // gin框架实例
        params  *Params             // URL参数切片

        mu sync.RWMutex //读写锁，用来保护Keys
	    Keys map[string]any //键是专门针对每个请求的上下文的键/值对
    }
```



## 路由注册流程


### gin.RouterGroup

什么是路由？就是根据不同的URL，找到不同的处理函数。