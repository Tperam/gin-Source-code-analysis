# gin框架刨析

## 路由处理

根据 golang原生服务器开发。我们得知对于服务器的请求都会交由一个实现`ServeHTTP(resWriter http.ResponseWriter, req *http.Request)`方法的结构体进行实现

gin框架也是基于此方法进行封装 `gin/gin.go:361`

### `ServeHTTP`源码

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

### `handleHTTPRequest`源码

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

### `methodTree`结构体

并且展开 树的结构体 `gin/tree.go`

```go
type methodTree struct {
	method string
	root   *node
}

type methodTrees []methodTree
```

暂时不对其进行讲解。



回顾我们的代码，如何创建一个路由 ( 从此处深入 )

```go
r.Post("/register",controller.MemberRegister)
```

### `POST`源码

我们再查看gin提供给我们的`Post`方法的源码 `gin/routergroup.go:97`

```go
// POST is a shortcut for router.Handle("POST", path, handle).
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodPost, relativePath, handlers)
}
```

### `handle`源码

发现他调用了`group.handle`函数。继续往下 `gin/routergroup.go:72`

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    // 记录当前绝对路径。
	absolutePath := group.calculateAbsolutePath(relativePath)
    // 连接处理方法
	handlers = group.combineHandlers(handlers)
    // 添加树节点
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
      if finalSize >= int(abortIndex) { // 判断处理方法是否超过最大数(math.MaxInt8 / 2)
  		panic("too many handlers")
  	}
      // 创建一个 handlersChain 这是一个type HandlerFunc func(*Context)的函数切片
  	mergedHandlers := make(HandlersChain, finalSize)
      // 浅度复制，将处理方法的指针复制到 mergedHandlers
  	copy(mergedHandlers, group.Handlers)
  	copy(mergedHandlers[len(group.Handlers):], handlers)
  	return mergedHandlers
  }
  ```

- `group.engine.addRoute` 当前方法为我们着重介绍的内容。

### `group.engine.addRoute`源码

  我们继续展开源码。`gin/gin.go:252`

  ```go
  func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
  	assert1(path[0] == '/', "path must begin with '/'") // 断言
  	assert1(method != "", "HTTP method can not be empty")
  	assert1(len(handlers) > 0, "there must be at least one handler")
  
  	debugPrintRoute(method, path, handlers)
  	root := engine.trees.get(method) // 获取指定树
  	if root == nil { // 如果为空
  		root = new(node)
  		root.fullPath = "/"
  		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
  	root.addRoute(path, handlers) // 添加树节点
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

  发现他是遍历当前切片中的method。如果method等于`http.MethodPost`则返回。如果没有我们所需要的内容则返回 nil。那么也就意味着每一个 method 至少是一棵树 `GET` `POST` `PUT` `DELETE`...

  我们继续返回到`group.engine.addRoute`方法的源码，其中8-12行为对根节点的初始化。

###   `root.addRoute`源码

  展开`root.addRoute(path, handlers)`方法，它在`gin/tree.go:133`

  ```go
func (n *node) addRoute(path string, handlers HandlersChain) {
    fullPath := path
    n.priority++
    numParams := countParams(path) // 用于记录路由参数中的个数 /user/:uid; numParams:= 1

    // Empty tree
    if len(n.path) == 0 && len(n.children) == 0 {
        n.insertChild(numParams, path, fullPath, handlers)
        n.nType = root
        return
    }

    parentFullPathIndex := 0

walk:
    for {
        // 判断是否大于最大路由参数个数。如果大于重新记录
        // Update maxParams of the current node
        if numParams > n.maxParams {
            n.maxParams = numParams
        }

        // 输入a字符串与b字符串 将会返回字符串 a[n]==b[n]的个数
        // 2 = longestCommonPrefix("ayye", "cywe")
        // Find the longest common prefix.
        // This also implies that the common prefix contains no ':' or '*'
        // since the existing key can't contain those chars.
        i := longestCommonPrefix(path, n.path)

        // Split edge
        if i < len(n.path) {
            child := node{
                path:      n.path[i:],
                wildChild: n.wildChild,
                indices:   n.indices,
                children:  n.children,
                handlers:  n.handlers,
                priority:  n.priority - 1,
                fullPath:  n.fullPath,
            }

            // Update maxParams (max of all children)
            for _, v := range child.children {
                if v.maxParams > child.maxParams {
                    child.maxParams = v.maxParams
                }
            }

            n.children = []*node{&child}
            // []byte for proper unicode char conversion, see #65
            n.indices = string([]byte{n.path[i]})
            n.path = path[:i]
            n.handlers = nil
            n.wildChild = false
            n.fullPath = fullPath[:parentFullPathIndex+i]
        }

        // Make new node a child of this node
        if i < len(path) {
            path = path[i:]

            if n.wildChild {
                parentFullPathIndex += len(n.path)
                n = n.children[0]
                n.priority++

                // Update maxParams of the child node
                if numParams > n.maxParams {
                    n.maxParams = numParams
                }
                numParams--

                // Check if the wildcard matches
                if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
                    // check for longer wildcard, e.g. :name and :names
                    if len(n.path) >= len(path) || path[len(n.path)] == '/' {
                        continue walk
                    }
                }

                pathSeg := path
                if n.nType != catchAll {
                    pathSeg = strings.SplitN(path, "/", 2)[0]
                }
                prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
                panic("'" + pathSeg +
                      "' in new path '" + fullPath +
                      "' conflicts with existing wildcard '" + n.path +
                      "' in existing prefix '" + prefix +
                      "'")
            }

            c := path[0]

            // slash after param
            if n.nType == param && c == '/' && len(n.children) == 1 {
                parentFullPathIndex += len(n.path)
                n = n.children[0]
                n.priority++
                continue walk
            }

            // Check if a child with the next path byte exists
            for i, max := 0, len(n.indices); i < max; i++ {
                if c == n.indices[i] {
                    parentFullPathIndex += len(n.path)
                    i = n.incrementChildPrio(i)
                    n = n.children[i]
                    continue walk
                }
            }

            // Otherwise insert it
            if c != ':' && c != '*' {
                // []byte for proper unicode char conversion, see #65
                n.indices += string([]byte{c})
                child := &node{
                    maxParams: numParams,
                    fullPath:  fullPath,
                }
                n.children = append(n.children, child)
                n.incrementChildPrio(len(n.indices) - 1)
                n = child
            }
            n.insertChild(numParams, path, fullPath, handlers)
            return
        }

        // Otherwise and handle to current node
        if n.handlers != nil {
            panic("handlers are already registered for path '" + fullPath + "'")
        }
        n.handlers = handlers
        return
    }
}
  ```

  - `n.insertChild(numParams, path, fullPath, handlers)`，用于对树插入节点。 `gin/tree.go:292`
  
    ```go
    func (n *node) insertChild(numParams uint8, path string, fullPath string, handlers HandlersChain) {
        // 判断路由参数是否大于0
        // 如果存在路由参数，则进入执行
    	for numParams > 0 {
    		// Find prefix until first wildcard
            // 当前方法返回三个参数。第一个为通配符的字符串，第二个为从第几个字符串出现的通配符，第三个为返回是否是符合规则的通配符
    		wildcard, i, valid := findWildcard(path)
    		if i < 0 { // No wildcard found
    			break
    		}
    
    		// The wildcard name must not contain ':' and '*'
    		if !valid {
    			panic("only one wildcard per path segment is allowed, has: '" +
    				wildcard + "' in path '" + fullPath + "'")
    		}
    
    		// check if the wildcard has a name
    		if len(wildcard) < 2 {
    			panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
    		}
    
    		// Check if this node has existing children which would be
    		// unreachable if we insert the wildcard here
    		if len(n.children) > 0 {
    			panic("wildcard segment '" + wildcard +
    				"' conflicts with existing children in path '" + fullPath + "'")
    		}
            // 如果通配符的第一个字符为 :
    		if wildcard[0] == ':' { // param
                // 将通配符前面的字符记录到当前node中的path
    			if i > 0 {
    				// Insert prefix before the current wildcard
    				n.path = path[:i]
    				path = path[i:]
    			}
    			// 生成子节点 并将其引向当前节点
    			n.wildChild = true
    			child := &node{
    				nType:     param,
    				path:      wildcard,
    				maxParams: numParams,
    				fullPath:  fullPath,
    			}
    			n.children = []*node{child}
    			// 更新n引用到子节点
    			n = child
    			n.priority++
    			numParams--
    
                // 当前if用于防止遗漏 如果不执行当前代码则无法遍历到 `/user/:uid/update`中的update
    			// 判断除去当前通配符后是否还有剩余路由
    			// if the path doesn't end with the wildcard, then there
    			// will be another non-wildcard subpath starting with '/'
    			if len(wildcard) < len(path) {
    				// 将path重置到当前通配符之后的字符串
    				path = path[len(wildcard):]
    				// 生成子节点 并将其引向当前节点
    				child := &node{
    					maxParams: numParams,
    					priority:  1,
    					fullPath:  fullPath,
    				}
                    
    				n.children = []*node{child}
    				n = child
    				continue
    			}
    
    			// Otherwise we're done. Insert the handle in the new leaf
    			n.handlers = handlers
    			return
    		}
    
    		// catchAll
    		if i+len(wildcard) != len(path) || numParams > 1 {
    			panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
    		}
    
    		if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
    			panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
    		}
    
    		// currently fixed width 1 for '/'
    		i--
    		if path[i] != '/' {
    			panic("no / before catch-all in path '" + fullPath + "'")
    		}
    
    		n.path = path[:i]
    
    		// First node: catchAll node with empty path
    		child := &node{
    			wildChild: true,
    			nType:     catchAll,
    			maxParams: 1,
    			fullPath:  fullPath,
    		}
    		// update maxParams of the parent node
    		if n.maxParams < 1 {
    			n.maxParams = 1
    		}
    		n.children = []*node{child}
    		n.indices = string('/')
    		n = child
    		n.priority++
    
    		// second node: node holding the variable
    		child = &node{
    			path:      path[i:],
    			nType:     catchAll,
    			maxParams: 1,
    			handlers:  handlers,
    			priority:  1,
    			fullPath:  fullPath,
    		}
    		n.children = []*node{child}
    
    		return
    	}
    
    	// If no wildcard was found, simply insert the path and handle
    	n.path = path
    	n.handlers = handlers
    	n.fullPath = fullPath
    }
    ```
### `findWildcard`源码

我们对当前方法进行源码展开 ，位置: `gin/tree.go:269`

```go
func findWildcard(path string) (wildcard string, i int, valid bool) {
    // Find start
    for start, c := range []byte(path) {
        // A wildcard starts with ':' (param) or '*' (catch-all)
        if c != ':' && c != '*' {
            continue
        }
        // 检查是否有效
        // Find end and check for invalid characters
        valid = true
        for end, c := range []byte(path[start+1:]) {
            switch c {
                case '/':
                // 当查找到 / 时返回通配符后的名字，以及通配符下标，以及当前命名是否有效
                return path[start : start+1+end], start, valid
                case ':', '*':
                valid = false
            }
        }
        return path[start:], start, valid
    }
    // 如果没有通配符则直接返回
	return "", -1, false
}
```
当前这个方法是用于获取通配符，通配符开始位置以及名字，在 `root.addRoute`中起到剥离作用

经过以上分析我们已经知道 当`methodTree`为空时我们是如何插入的，也大概了解到了`node`结构体中的属性

我们现在展开 `node` 结构体进行分析 `gin/tree.go:96`

```go
type node struct {
	path      string  // 当前节点路径
	indices   string
	children  []*node // 子节点列表
	handlers  HandlersChain // 当前路径的处理方法
	priority  uint32 // 优先级 
	nType     nodeType // 节点类型 ，root 根节点,param 路由参数节点, catchAll ??, static ??
	maxParams uint8 // 最大路由参数
	wildChild bool // 是否有动态子路由
	fullPath  string // 绝对路径
}
```

现在，我们开始模拟运行一遍这行代码 [link](post树理解.md)

```go
r.POST("/register",controller.MemberRegister)
r.POST("/login",controller.MemberLogin)
```

思考他进行了哪些操作 