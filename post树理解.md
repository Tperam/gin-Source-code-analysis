# 解析添加一个Post请求处理

## 第一次添加POST请求处理

语句：

```go
r.POST("/register",controller.MemberRegister) 
```

其中 `controller.MemberRegister`为自定义controller

当前方法调用了 `group.handle`

```go
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc) IRoutes {
    return group.handle(http.MethodPost, relativePath, handlers)
}
```

group.handle 方法调用了

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	// 获取当前的绝对路径
    absolutePath := group.calculateAbsolutePath(relativePath)
    // 由于我们使用的是默认路由组，所以他当前组的path为 '/'
    // 所以结果为 /register
    
    // 将所有处理方法连成一个切片
	handlers = group.combineHandlers(handlers)
    // 默认 gin.Default()会附带两个方法 Logger(), Recovery()
    // 所以现在方法为 Logger,Recovery,controller.MemberRegister
    
    // 添加路由
    // 传入值
    // Method = "POST"
    // absolutePath = "/register"
    // handlers = HandlersChain  --> []HandlerFunc --> [] func(*gin.Context)
    // handlers = []HandlerFunc{ Logger,Recovery,controller.MemberRegister }
	group.engine.addRoute(httpMethod, absolutePath, handlers)
    //
	return group.returnObj()
}
```

进入关键方法 `group.engine.addRoute(httpMethod, absolutePath, handlers)`

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    // 断言
    // 不以 / 开头则返回
	assert1(path[0] == '/', "path must begin with '/'")
    // 没有 http method
	assert1(method != "", "HTTP method can not be empty")
    // 处理方法少于1
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)
    // 获取方法树
    // POST 请求方法有一个 radix树
    // GET 请求方法也有方法数
    // ...
	root := engine.trees.get(method)
    // 判断是否是空树
	if root == nil {
        // 创建一个节点
		root = new(node)
        // 完整路径
		root.fullPath = "/"
        // 将添加当前 POST 方法的树
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
    // 添加 路径以及节点
    // 传入值
    // path = "/register"
    // handlers = []HandlerFunc{ Logger,Recovery,controller.MemberRegister }
	root.addRoute(path, handlers)
}
```

进入`root.addRoute(path,handlers)`

```go
func (n *node) addRoute(path string, handlers HandlersChain) {
    // fullPath = "/register"
	fullPath := path
    // n.priority = 1
	n.priority++
    // numParams = 0
	numParams := countParams(path)
	
    // 我们当前是空树。所以执行里面方法
	// Empty tree
	if len(n.path) == 0 && len(n.children) == 0 {
        // 插入子节点
		n.insertChild(numParams, path, fullPath, handlers)
        // 设置当前n = root
		n.nType = root
        // 结束方法
		return
	}

	parentFullPathIndex := 0

walk:
	for {
		// Update maxParams of the current node
		if numParams > n.maxParams {
			n.maxParams = numParams
		}

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

## 第二次添加POST请求处理

语句：

```go
r.POST("/login",controller.MemberLogin) // 
```

其中 `controller.MemberLogin`为自定义controller