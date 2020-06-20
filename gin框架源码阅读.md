# gin框架刨析

## 路由处理

根据 golang原生服务器开发。我们得知对于服务器的请求都会交由一个实现`ServeHTTP(resWriter http.ResponseWriter, req *http.Request)`方法的结构体进行实现

gin框架也是基于此方法进行封装 `gin/gin.go:361`

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```

在`ServeHTTP`方法中调用了`handleHTTPRequest()`，当前方法命名十分直观，用于处理`http`请求


我们对其进行源码展开`gin/gin.go:383`

```go
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

	// Find root of the tree for the given HTTP method
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// Find route in tree
		value := root.getValue(rPath, c.Params, unescape)
		if value.handlers != nil {
			c.handlers = value.handlers
			c.Params = value.params
			c.fullPath = value.fullPath
			c.Next()
			c.writermem.WriteHeaderNow()
			return
		}
		if httpMethod != "CONNECT" && rPath != "/" {
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
			if value := tree.root.getValue(rPath, nil, unescape); value.handlers != nil {
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

直接从14行开始看。假定gin是通过这个`t := engine.trees`去查找对于的路由。那么我们就开始思考，他怎么添加节点到树的。

并且展开 树的结构体 `gin/tree.go`

```go
type methodTree struct {
	method string
	root   *node
}

type methodTrees []methodTree
```

暂时不对其进行讲解。



回顾我们的代码，如何创建一个路由

```go
r.Post("/register",controller.MemberRegister)
```

我们再查看gin提供给我们的`Post`方法的源码 `gin/routergroup.go:97`

```go
// POST is a shortcut for router.Handle("POST", path, handle).
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}
```

发现他调用了`group.handle`函数。继续往下 `gin/routergroup.go:72`

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
```

对当前代码进行一个简单分析

- `absolutePath` 单看命名可得知 当前变量是用于记录绝对路径的。我们通过查看`group.calculateAbsolutePath`方法验证了我们的结论。该方法返回 组路径与当前路径的拼接

- `handlers` 单看命名可得知 当前变量适用于记录处理方法的。我们通过查看`group.combineHandlers`方法验证了当前结论。该方法讲当前组中的`handlers`方法与当前路由的`handlers`方法进行了拷贝。并将其以切片的形式进行返回  `gin/routergroup.go:210`

  ```go
  func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
  	finalSize := len(group.Handlers) + len(handlers)
  	if finalSize >= int(abortIndex) {
  		panic("too many handlers")
  	}
  	mergedHandlers := make(HandlersChain, finalSize)
  	copy(mergedHandlers, group.Handlers)
  	copy(mergedHandlers[len(group.Handlers):], handlers)
  	return mergedHandlers
  }
  ```

- `group.engine.addRoute` 当前方法为我们着重介绍的内容。我们继续展开源码。`gin/gin.go:252`

  ```go
  func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
  	assert1(path[0] == '/', "path must begin with '/'")
  	assert1(method != "", "HTTP method can not be empty")
  	assert1(len(handlers) > 0, "there must be at least one handler")
  
  	debugPrintRoute(method, path, handlers)
  	root := engine.trees.get(method)
  	if root == nil {
  		root = new(node)
  		root.fullPath = "/"
  		engine.trees = append(engine.trees, methodTree{method: method, root: root})
  	}
  	root.addRoute(path, handlers)
  }
  ```

  `assert1` 断言。用于阻止不符合规定的代码进入。

  我们从第七行开始看。

  发现他调用了 engine中的trees变量。现在我们去查看他的类型`gin/gin.go:114`

  ```go
  type Engine Struct{
  	trees            methodTrees
  }
  ```

  发现他是我们开头所提到`methodTrees`树类型。

  我们查看其中的方法 `gin/tree.go:49`

  ```go
  func (trees methodTrees) get(method string) *node {
  	for _, tree := range trees {
  		if tree.method == method {
  			return tree.root
  		}
  	}
  	return nil
  }
  ```

  发现他是遍历当前切片中的method。如果method等于`http.MethodPost`则返回。如果没有我们所需要的内容则返回 nil

  我们继续返回到`group.engine.addRoute`方法的源码，其中8-12行为对根节点的初始化。

  展开`root.addRoute`方法，它在`gin/tree.go`

  

  好的，有点太多不想展开。

  