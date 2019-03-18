---
title: 服务计算 | gorilla/mux 源码解析
date: 2018-11-15 19:29:55
categories:
  - Go
tags: 
  - Go
---

# gorilla/mux 源码解析
在开始阅读 `gorilla/mux` 源码之前，不妨让我们看看 `Golang` 的 `net/http` 包是怎么实现多路复用的。

## net/http
用一个简单的例子过一下流程:
```go
// Hello world, the web server
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	helloHandler := func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "Hello, world!\n")
	}

	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

首先我们调用了 `http.HandleFunc` 来为 `/hello` 注册一个 `Handler`：
```go
http.HandleFunc("/hello", helloHandler)
```

依次调用下面三个函数/方法，第一个函数表明当我们使用 http.HandleFunc 时默认是注册在 `DefaultServeMux` 这个默认的多路复用器上；第二个方法中 `HandlerFunc(handler)` 意思是将 `handler` 强制转化为 `HandlerFunc` 因为其实现了 `http.Handler` 这个接口；第三个方法在 mux.m 中查找是否已为该 `URL` 注册 `handler` ，如果有就报错，没有就为其注册：
```go
// ServeMux *
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}

// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
} 

// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}

// Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

接下来是调用：
```go
http.ListenAndServe(":8080", nil)
```

本文不打算追踪启动 `server` 的整个过程，不了解或者感兴趣的可以看看 [Go的http包详解](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/03.4.md) 。我们只挑对我们讲解多路复用感兴趣的部分来讲解，下面看看：
```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```
上面的方法表明当我们在调用 `http.ListenAndServe(":8080", nil)` 没有给 `handler` 的时候，默认使用 `DefaultServeMux` ，所以上面的例子调用的是 `DefaultServeMux` 的 `ServeHTTP`：
```go
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

让我们看看 `DefaultServeMux` 是如何查找相应的 `handler` ，看看 `match` 的实现，只能简单地实现字符串匹配，用起来不够爽吧？
```go
// Find a handler on a handler map given a path string.
// Most-specific (longest) pattern wins.
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.
	var n = 0
	for k, v := range mux.m {
		if !pathMatch(k, path) {
			continue
		}
		if h == nil || len(k) > n {
			n = len(k)
			h = v.h
			pattern = v.pattern
		}
	}
	return
}
```

当然，复杂也可能意味着性能下降，有兴趣的同学可以看看 [Go HTTP Router Benchmark](https://github.com/julienschmidt/go-http-routing-benchmark/blob/master/README.md) ，毕竟系统是拿来用的，不只是拿来写的。不过作为一个合格的程序员，应该学会用轮子，所以让我们看看 `gorilla/mux` 是怎么实现路由的吧。

## gorilla/mux
一个简单使用的例子（摘自[官方文档](https://github.com/gorilla/mux#graceful-shutdown)）：
```go
package main

import (
    "net/http"
    "log"
    "github.com/gorilla/mux"
)

func YourHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Gorilla!\n"))
}

func main() {
    r := mux.NewRouter()
    // Routes consist of a path and a handler function.
    r.HandleFunc("/", YourHandler)

    // Bind to a port and pass our router in
    log.Fatal(http.ListenAndServe(":8000", r))
}
```

一些进阶用法：
```go
// 可以匹配 URL 中的变量
r := mux.NewRouter()
r.HandleFunc("/products/{key}", ProductHandler)
r.HandleFunc("/articles/{category}/", ArticlesCategoryHandler)
r.HandleFunc("/articles/{category}/{id:[0-9]+}", ArticleHandler)

// 使用匹配的变量
func ArticlesCategoryHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    w.WriteHeader(http.StatusOK)
    fmt.Fprintf(w, "Category: %v\n", vars["category"])
}

// 路径前缀
r.PathPrefix("/products/")

// Only matches if domain is "www.example.com".
r.Host("www.example.com")
// Matches a dynamic subdomain.
r.Host("{subdomain:[a-z]+}.domain.com")

// 子路由
r := mux.NewRouter()
s := r.Host("www.example.com").Subrouter()
```
ok，总之它的功能比官方包要强大的多，下面开始源码分析。

### mux.go
先看看其数据结构，与 `http.ServeMux` 相比，要多出不少东西，我们主要关注 `routes` 、`parent` 以及三个 `flag`：`strictSlash`、`skipClean`、`useEncodedPath`。
```go
// This will send all incoming requests to the router.
type Router struct {
	// Configurable Handler to be used when no route matches.
	NotFoundHandler http.Handler

	// Configurable Handler to be used when the request method does not match the route.
	MethodNotAllowedHandler http.Handler

	// Parent route, if this is a subrouter.
	parent parentRoute
	// Routes to be matched, in order.
	routes []*Route
	// Routes by name for URL building.
	namedRoutes map[string]*Route
	// See Router.StrictSlash(). This defines the flag for new routes.
	strictSlash bool
	// See Router.SkipClean(). This defines the flag for new routes.
	skipClean bool
	// If true, do not clear the request context after handling the request.
	// This has no effect when go1.7+ is used, since the context is stored
	// on the request itself.
	KeepContext bool
	// see Router.UseEncodedPath(). This defines a flag for all routes.
	useEncodedPath bool
	// Slice of middlewares to be called after a match is found
	middlewares []middleware
}
```

其中三个 `flag` 只要看注释就知道它们的作用了，不多解释：
* strictSlash
    ```go
    // StrictSlash defines the trailing slash behavior for new routes. The initial
    // value is false.
    //
    // When true, if the route path is "/path/", accessing "/path" will perform a redirect
    // to the former and vice versa. In other words, your application will always
    // see the path as specified in the route.
    //
    // When false, if the route path is "/path", accessing "/path/" will not match
    // this route and vice versa.
    func (r *Router) StrictSlash(value bool) *Router {
        r.strictSlash = value
        return r
    }
    ```

* skipClean
    ```go
    // SkipClean defines the path cleaning behaviour for new routes. The initial
    // value is false. Users should be careful about which routes are not cleaned
    //
    // When true, if the route path is "/path//to", it will remain with the double
    // slash. This is helpful if you have a route like: /fetch/http://xkcd.com/534/
    //
    // When false, the path will be cleaned, so /fetch/http://xkcd.com/534/ will
    // become /fetch/http/xkcd.com/534
    func (r *Router) SkipClean(value bool) *Router {
        r.skipClean = value
        return r
    }
    ```

* useEncodedPath
    ```go
    // UseEncodedPath tells the router to match the encoded original path
    // to the routes.
    // For eg. "/path/foo%2Fbar/to" will match the path "/path/{var}/to".
    //
    // If not called, the router will match the unencoded path to the routes.
    // For eg. "/path/foo%2Fbar/to" will match the path "/path/foo/bar/to"
    func (r *Router) UseEncodedPath() *Router {
        r.useEncodedPath = true
        return r
    }
    ```

先看看它的 `HandleFunc` ，首先 `NewRoute` 创建一个 `Route` ，然后 `Path` 为 `path` 添加一个 `matcher` ，最后 `HandlerFunc` 为其设置一个 `handler` ：
```go
// HandleFunc registers a new route with a matcher for the URL path.
// See Route.Path() and Route.HandlerFunc().
func (r *Router) HandleFunc(path string, f func(http.ResponseWriter,
	*http.Request)) *Route {
	return r.NewRoute().Path(path).HandlerFunc(f)
}
```

其次看看 `Router` 的 `ServeHTTP` ，先使用上面说的 `flag` 对 `Request` 中的 `URL` 进行预处理，然后使用 `Router` 的 `Match` 方法查找 `handler`。
```go
// ServeHTTP dispatches the handler registered in the matched route.
//
// When there is a match, the route variables can be retrieved calling
// mux.Vars(request).
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	if !r.skipClean {
		path := req.URL.Path
		if r.useEncodedPath {
			path = req.URL.EscapedPath()
		}
		// Clean path to canonical form and redirect.
		if p := cleanPath(path); p != path {

			// Added 3 lines (Philip Schlump) - It was dropping the query string and #whatever from query.
			// This matches with fix in go 1.2 r.c. 4 for same problem.  Go Issue:
			// http://code.google.com/p/go/issues/detail?id=5252
			url := *req.URL
			url.Path = p
			p = url.String()

			w.Header().Set("Location", p)
			w.WriteHeader(http.StatusMovedPermanently)
			return
		}
	}
	var match RouteMatch
	var handler http.Handler
	if r.Match(req, &match) {
		handler = match.Handler
		req = setVars(req, match.Vars)
		req = setCurrentRoute(req, match.Route)
	}

	if handler == nil && match.MatchErr == ErrMethodMismatch {
		handler = methodNotAllowedHandler()
	}

	if handler == nil {
		handler = http.NotFoundHandler()
	}

	if !r.KeepContext {
		defer contextClear(req)
	}

	handler.ServeHTTP(w, req)
}
```

再看看 `Router` 的 `Match` 方法，遍历 `Router` 下的所有 `route` 看看是否匹配：
```go
// Match attempts to match the given request against the router's registered routes.
//
// If the request matches a route of this router or one of its subrouters the Route,
// Handler, and Vars fields of the the match argument are filled and this function
// returns true.
//
// If the request does not match any of this router's or its subrouters' routes
// then this function returns false. If available, a reason for the match failure
// will be filled in the match argument's MatchErr field. If the match failure type
// (eg: not found) has a registered handler, the handler is assigned to the Handler
// field of the match argument.
func (r *Router) Match(req *http.Request, match *RouteMatch) bool {
	for _, route := range r.routes {
		if route.Match(req, match) {
			// Build middleware chain if no error was found
			if match.MatchErr == nil {
				for i := len(r.middlewares) - 1; i >= 0; i-- {
					match.Handler = r.middlewares[i].Middleware(match.Handler)
				}
			}
			return true
		}
	}
  // 省略
	...
}
```

在这里我们还不能看出 `Router` 是怎么实现如此强大的路由的，因为重点是 Route。

### route.go
照例看看数据结构，有父路由 `parent` 、处理器 `handler` 、匹配器表 `matchers` 、正则表达式 `regexp` 以及前面提到的三个 `flag` 等：
```go
// Route stores information to match a request and build URLs.
type Route struct {
	// Parent where the route was registered (a Router).
	parent parentRoute
	// Request handler for the route.
	handler http.Handler
	// List of matchers.
	matchers []matcher
	// Manager for the variables from host and path.
	regexp *routeRegexpGroup
	// If true, when the path pattern is "/path/", accessing "/path" will
	// redirect to the former and vice versa.
	strictSlash bool
	// If true, when the path pattern is "/path//to", accessing "/path//to"
	// will not redirect
	skipClean bool
	// If true, "/path/foo%2Fbar/to" will match the path "/path/{var}/to"
	useEncodedPath bool
	// The scheme used when building URLs.
	buildScheme string
	// If true, this route never matches: it is only used to build URLs.
	buildOnly bool
	// The name used to build URLs.
	name string
	// Error resulted from building a route.
	err error

	buildVarsFunc BuildVarsFunc
}
```

还记得前面创建 `Route` 后使用的 `Path` 方法吧，它能为 `URL path` 添加一个正则表达式匹配器：
```go
// Path adds a matcher for the URL path.
// It accepts a template with zero or more URL variables enclosed by {}. The
// template must start with a "/".
// Variables can define an optional regexp pattern to be matched:
//
// - {name} matches anything until the next slash.
//
// - {name:pattern} matches the given regexp pattern.
//
// For example:
//
//     r := mux.NewRouter()
//     r.Path("/products/").Handler(ProductsHandler)
//     r.Path("/products/{key}").Handler(ProductsHandler)
//     r.Path("/articles/{category}/{id:[0-9]+}").
//       Handler(ArticleHandler)
//
// Variable names must be unique in a given route. They can be retrieved
// calling mux.Vars(request).
func (r *Route) Path(tpl string) *Route {
	r.err = r.addRegexpMatcher(tpl, regexpTypePath)
	return r
}
```

这里就不打算再深入下去探究如何添加的啦，一来作者水平有限，二来大家有不同的实现方式，所以我们就且停留到这一层吧。下面看看 `Route` 的 `Match` ：
```go
// Match matches the route against the request.
func (r *Route) Match(req *http.Request, match *RouteMatch) bool {
	if r.buildOnly || r.err != nil {
		return false
	}

	var matchErr error

	// Match everything.
	for _, m := range r.matchers {
		if matched := m.Match(req, match); !matched {
			if _, ok := m.(methodMatcher); ok {
				matchErr = ErrMethodMismatch
				continue
			}
			matchErr = nil
			return false
		}
	}

	if matchErr != nil {
		match.MatchErr = matchErr
		return false
	}

	if match.MatchErr == ErrMethodMismatch {
		// We found a route which matches request method, clear MatchErr
		match.MatchErr = nil
		// Then override the mis-matched handler
		match.Handler = r.handler
	}

	// Yay, we have a match. Let's collect some info about it.
	if match.Route == nil {
		match.Route = r
	}
	if match.Handler == nil {
		match.Handler = r.handler
	}
	if match.Vars == nil {
		match.Vars = make(map[string]string)
	}

	// Set variables.
	if r.regexp != nil {
		r.regexp.setMatch(req, match, r)
	}
	return true
}
```

到这里大家可能会疑惑了，`// Match everything` 指的是什么，其实一个 `Route` 可以存在多个 `matcher`，也就是说，你可能设置了一个这样的路由:
```go
r.HandleFunc("/products", ProductsHandler).
  Host("www.example.com").
  Methods("GET").
  Schemes("http")
```

那么这个路由就需要满足上述 4 个 `matcher` 才能够匹配成功，而 `Path` 和 `Host` 都是通过添加 `routeRegexp` 来实现，`Methods` 和 `Schemes` 则分别实现了各自的 `matcher`：
```go
// Path and Host add RegexpMatcher
func (r *Route) Path(tpl string) *Route {
	r.err = r.addRegexpMatcher(tpl, regexpTypePath)
	return r
}

func (r *Route) Host(tpl string) *Route {
	r.err = r.addRegexpMatcher(tpl, regexpTypeHost)
	return r
}

// methodMatcher matches the request against HTTP methods.
type methodMatcher []string

func (m methodMatcher) Match(r *http.Request, match *RouteMatch) bool {
	return matchInArray(m, r.Method)
}


// schemeMatcher matches the request against URL schemes.
type schemeMatcher []string

func (m schemeMatcher) Match(r *http.Request, match *RouteMatch) bool {
	return matchInArray(m, r.URL.Scheme)
}
```

## 总结
你有可能看了我的分析以后更晕了，但是没关系，你只要记住 `gorilla/mux` 大致是怎么实现路由的就好：
1. 使用 `Router` 的 `HandleFunc` 、`Host` 、 `Methods` 、 `Schemes` 、 `Headers` 等方法会创建一个路由并且为其添加相应类型 `matcher`，当然你也可以使用 `MatcherFunc` 来创建自己的 `matcher`。如果你想为一个路由添加多个方法，那么你可以这样：
    ```go
    r.HandleFunc("/products", ProductsHandler).
    Host("www.example.com").
    Methods("GET").
    Schemes("http)
    ```
    因为上面说的每个方法都会返回一个刚创建 Route 的指针，这样你就可以使用 Route 的同名方法来添加 matcher 了。

2. `gorilla/mux` 会查找 `Router` 下的 `Route` 列表，找到是否有无匹配的路由，而 `Route` 会遍历其 `matchers` 列表来看看是否满足所有的 `matchers`，只要一个不匹配，则失败。