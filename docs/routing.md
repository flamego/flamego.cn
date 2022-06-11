---
prev:
  text: 自定义服务
  link: custom-services
next:
  text: 中间件集成
  link: middleware
---

# 路由配置

每一个来自客户端的请求都会经过路由系统，因此路由系统的易用性对于一个 Web 框架来说是至关重要的。Flamego 在路由系统的设计和实现上花费了大量精力，在确保易用性的同时保留了未来的可扩展性。

路由是指 HTTP 请求方法和 URL 匹配模式的组合，并且每个路由都可以绑定多个处理器。

下面列举了对应不同 HTTP 请求的辅助方法：

```go:no-line-numbers
f.Get("/", ...)
f.Patch("/", ...)
f.Post("/", ...)
f.Put("/", ...)
f.Delete("/", ...)
f.Options("/", ...)
f.Head("/", ...)
f.Connect("/", ...)
f.Trace("/", ...)
```

`Any` 方法可以将单个路由与所有 HTTP 请求方法进行组合：

```go:no-line-numbers
f.Any("/", ...)
```

当你需要将单个路由与多个 HTTP 请求方法进行组合时，则可以使用 `Routes` 方法：

```go:no-line-numbers
f.Routes("/", "GET,POST", ...)

// 或者

f.Routes("/", http.MethodGet, http.MethodPost, ...)
```

## 术语

- **URL 路径块**是指介于两个斜杠之间的部分，如 `/<segment>/`，且尾部的斜杠可以被省略
- **绑定参数** 使用花括号（`{}`）进行表示，并仅限用于[动态路由](#动态路由)

## 静态路由

静态路由是 Web 应用中最为常见的一种路由，即要求客户端的请求路径与配置的路径完整一致才能被匹配：

```go:no-line-numbers
f.Get("/user", ...)
f.Get("/repo", ...)
```

上例中，任何不为 `/user` 或 `/repo` 的请求路径都将收到 404。

::: warning
标准库的 `net/http` 允许用户使用 `/user/` 来匹配所有以 `/user` 开头的请求路径，但在 Flamego 中这仍旧只是一个单纯的静态路由，所以要求客户端的请求路径与 `/user/` 完全一致才能被匹配。

来看个例子就明白了：

```go:no-line-numbers
package main

import (
	"github.com/flamego/flamego"
)

func main() {
	f := flamego.New()
	f.Get("/user/", func() string {
		return "You got it!"
	})
	f.Run()
}
```

运行如下测试：

```:no-line-numbers
$ curl http://localhost:2830/user
404 page not found

$ curl http://localhost:2830/user/
You got it!

$ curl http://localhost:2830/user/info
404 page not found
```
:::

## 动态路由

顾名思义，动态路由指的是可以进行动态匹配的路由配置。在撰写本文档时，Flamego 的动态路由在整个 Go 语言生态中依然首屈一指，无人望其项背。

`flamego.Context` 提供了一系列的辅助方法来获取动态路由中的绑定参数，包括：

- `Params` 返回所有的绑定参数
- `Param` 返回指定的绑定参数值
- `ParamInt` 返回解析为 `int` 类型的绑定参数值
- `ParamInt64` 返回解析为 `int64` 类型的绑定参数值

### 占位符

占位符可以用于匹配除了斜杠（`/`）以外的所有字符，并且在单个 URL 路径块中可以使用任意多个占位符。

下面列举了一些占位符的用法：

```go
f.Get("/users/{name}", ...)
f.Get("/posts/{year}-{month}-{day}.html", ...)
f.Get("/geo/{state}/{city}", ...)
```

在第 1 行，名为 `{name}` 的占位符会匹配整个 URL 路径块。

在第 2 行，`{year}`、`{month}` 和 `{day}` 这三个占位符会分别匹配 URL 路径块的三个部分。

在第 3 行，两个占位符由于在不同的 URL 路径块中，因此相互独立不受影响。

来看几个完整的例子：

:::: code-group
::: code-group-item 代码
```go:no-line-numbers
package main

import (
	"fmt"
	"strings"

	"github.com/flamego/flamego"
)

func main() {
	f := flamego.New()
	f.Get("/users/{name}", func(c flamego.Context) string {
		return fmt.Sprintf("The user is %s", c.Param("name"))
	})
	f.Get("/posts/{year}-{month}-{day}.html", func(c flamego.Context) string {
		return fmt.Sprintf(
			"The post date is %d-%d-%d",
			c.ParamInt("year"), c.ParamInt("month"), c.ParamInt("day"),
		)
	})
	f.Get("/geo/{state}/{city}", func(c flamego.Context) string {
		return fmt.Sprintf(
			"Welcome to %s, %s!",
			strings.Title(c.Param("city")),
			strings.ToUpper(c.Param("state")),
		)
	})
	f.Run()
}
```
:::
::: code-group-item 测试
```:no-line-numbers
$ curl http://localhost:2830/users/joe
The user is joe

$ curl http://localhost:2830/posts/2021-11-26.html
The post date is 2021-11-26

$ curl http://localhost:2830/geo/ma/boston
Welcome to Boston, MA!
```
:::
::::

::: tip
尝试执行 `curl http://localhost:2830/posts/2021-11-abc.html` 并观察输出的变化。
:::

### 正则表达式

正则表达式可以被用来进一步限定绑定参数的匹配规则，并使用斜杠进行表示，如 `/<regexp>/`。

下面列举了一些使用正则表达式定义的绑定参数：

```go
f.Get("/users/{name: /[a-zA-Z0-9]+/}", ...)
f.Get("/posts/{year: /[0-9]{4}/}-{month: /[0-9]{2}/}-{day: /[0-9]{2}/}.html", ...)
f.Get("/geo/{state: /[A-Z]{2}/}/{city}", ...)
```

在第 1 行，名为 `{name}` 的占位符会匹配整个 URL 路径块中的字母和数字。

在第 2 行，`{year}`、`{month}` 和 `{day}` 这三个占位符会分别匹配 URL 路径块中具有特定长度的数字。

在第 3 行，`{state}` 仅会匹配长度为 2 的大写字母。

::: tip
由于正则表达式自身是通过斜杠进行表示的，因此它们匹配规则不可以包含斜杠：

```:no-line-numbers
panic: unable to parse route "/{name: /abc\\//}": 1:15: unexpected token "/" (expected "}")
```
:::

来看几个完整的例子：

:::: code-group
::: code-group-item 代码
```go:no-line-numbers
package main

import (
	"fmt"
	"strings"

	"github.com/flamego/flamego"
)

func main() {
	f := flamego.New()
	f.Get("/users/{name: /[a-zA-Z0-9]+/}",
		func(c flamego.Context) string {
			return fmt.Sprintf("The user is %s", c.Param("name"))
		},
	)
	f.Get("/posts/{year: /[0-9]{4}/}-{month: /[0-9]{2}/}-{day: /[0-9]{2}/}.html",
		func(c flamego.Context) string {
			return fmt.Sprintf(
				"The post date is %d-%d-%d",
				c.ParamInt("year"), c.ParamInt("month"), c.ParamInt("day"),
			)
		},
	)
	f.Get("/geo/{state: /[A-Z]{2}/}/{city}",
		func(c flamego.Context) string {
			return fmt.Sprintf(
				"Welcome to %s, %s!",
				strings.Title(c.Param("city")),
				c.Param("state"),
			)
		},
	)
	f.Run()
}
```
:::
::: code-group-item 测试
```:no-line-numbers
$ curl http://localhost:2830/users/joe
The user is joe

$ curl http://localhost:2830/posts/2021-11-26.html
The post date is 2021-11-26

$ curl http://localhost:2830/geo/MA/boston
Welcome to Boston, MA!
```
:::
::::

::: tip
尝试运行以下测试并观察输出的变化：

```:no-line-numbers
$ curl http://localhost:2830/users/logan-smith
$ curl http://localhost:2830/posts/2021-11-abc.html
$ curl http://localhost:2830/geo/ma/boston
```
:::

### 通配符

使用通配符定义的绑定参数可以匹配多个 URL 路径块（包括斜杠）。通配符使用 `**` 进行表示，并接受一个可选参数 `capture` 用于设定最多可匹配 URL 路径块的数量。

下面列举了一些使用通配符定义的绑定参数：

```go
f.Get("/posts/{**}", ...) // "{**: **}" 的语法糖
f.Get("/webhooks/{repo: **}/events", ...)
f.Get("/geo/{**: **, capture: 2}", ...)
```

在第 1 行，通配符会匹配所有以 `/posts/` 开头的路径。

在第 2 行，通配符会匹配所有以 `/webhooks/` 开头并以 `/events` 结尾的路径。

在第 3 行，通配符会匹配所有以 `/geo/` 开头的路径，但最多匹配 2 个 URL 路径块。

来看几个完整的例子：

:::: code-group
::: code-group-item 代码
```go:no-line-numbers
package main

import (
	"fmt"
	"strings"

	"github.com/flamego/flamego"
)

func main() {
	f := flamego.New()
	f.Get("/posts/{**}",
		func(c flamego.Context) string {
			return fmt.Sprintf("The post is %s", c.Param("**"))
		},
	)
	f.Get("/webhooks/{repo: **}/events",
		func(c flamego.Context) string {
			return fmt.Sprintf("The event is for %s", c.Param("repo"))
		},
	)
	f.Get("/geo/{**: **, capture: 2}",
		func(c flamego.Context) string {
			fields := strings.Split(c.Param("**"), "/")
			return fmt.Sprintf(
				"Welcome to %s, %s!",
				strings.Title(fields[1]),
				strings.ToUpper(fields[0]),
			)
		},
	)
	f.Run()
}
```
:::
::: code-group-item 测试
```:no-line-numbers
$ curl http://localhost:2830/posts/2021/11/26.html
The post is 2021-11-26.html

$ curl http://localhost:2830/webhooks/flamego/flamego/events
The event is for flamego/flamego

$ curl http://localhost:2830/geo/ma/boston
Welcome to Boston, MA!
```
:::
::::

::: tip
尝试运行以下测试并观察输出的变化：

```:no-line-numbers
$ curl http://localhost:2830/webhooks/flamego/flamego
$ curl http://localhost:2830/geo/ma/boston/02125
```
:::

## 组合路由

当不同的 HTTP 方法需要与相同的一个路由进行组合时，可以使用 `Combo` 方法进行简写：

```go:no-line-numbers
f.Combo("/").Get(...).Post(...)
```

## 组路由

通过分组的方式对路由进行管理可以有效提升代码的可读性和中间件的复用。使用 `Group` 方法将可以将多个路由进行分组，分组内还可以嵌套更多的分组，并且嵌套的层数是没有限制的：

```go{4}
f.Group("/user", func() {
    f.Get("/info", ...)
    f.Group("/settings", func() {
        f.Get("", ...)
        f.Get("/account_security", ...)
    }, middleware3)
}, middleware1, middleware2)
```

在第 4 行的路径使用了空字符串，这是完全合法的使用方法，其等价于下面的配置：

```go:no-line-numbers
f.Get("/user/settings", ...)
```

![how does that work](https://media0.giphy.com/media/2gUHK3J2NCMsqsz1su/200w.webp?cid=ecf05e47d3syetfd9ja7nr3qwjfdrs4mnhjh46xq1numt01p&rid=200w.webp&ct=g)

这是因为 Flamego 的路由系统对组路由内的子路由使用[字符串拼接的方式来计算最终路径](https://github.com/flamego/flamego/blob/503ddd0f43a7281b5d334173aba5e32de2d2b31f/router.go#L201-L205)。

因此，下面用法也是合法的：

```go:no-line-numbers{3-5}
f.Group("/user", func() {
    f.Get("/info", ...)
    f.Group("/sett", func() {
        f.Get("ings", ...)
        f.Get("ings/account_security", ...)
    }, middleware3)
}, middleware1, middleware2)
```

## 可选路由

静态路由和动态路由均可被配置成可选路由，其使用问号（`?`）进行表示：

```go:no-line-numbers
f.Get("/user/?settings", ...)
f.Get("/users/?{name}", ...)
```

上面的用法等价的配置如下：

```go:no-line-numbers
f.Get("/user", ...)
f.Get("/user/settings", ...)
f.Get("/users", ...)
f.Get("/users/{name}", ...)
```

::: warning
可选路由只可被用于匹配最后一个 URL 路径块，并仅限在单个路由上配置一次。
:::

## 匹配请求头

::: tip 🆕 v1.5.0 版本新增
:::

你可以在匹配请求路径的基础上要求某个路由还需要匹配相应的请求头：

```go:no-line-numbers
f.Get("/", ...).Headers(
	"User-Agent", "Chrome",   // 宽松匹配
	"Host", "^flamego\.dev$", // 精准匹配
	"Cache-Control", "",      // 只要 "Cache-Control" 不为空
)
```

`Headers` 方法接受用于表示匹配请求头的键值对列表，键名对应请求头的名称，键值则是用于匹配的正则表达式。

当某个路由在请求头匹配环节失败时，Flame 实例会继续尝试匹配其它路由而不会中断匹配流程。

## 匹配优先级

随着 Web 应用的复杂化，配置的路由也会越来越多，此时对于路由之间匹配优先级的理解就显得至关重要。

匹配优先级是基于不同的 URL 匹配模式、匹配范围（范围越小优先级越高）和注册顺序决定的，具体如下：

1. 静态路由总是被优先匹配，如 `/users/settings`
1. 不匹配整个 URL 路径块的占位符，如 `/users/{name}.html`
1. 匹配整个 URL 路径块的占位符，如 `/users/{name}`.
1. 匹配中间路径的通配符，如 `/users/{**}/events`.
1. 匹配剩余路径的通配符，如 `/users/{**}`.

## 构建 URL 路径

`URLPath` 方法可以根据路由的名称构建其完整的路径：

```go:no-line-numbers
f.Get("/user/?settings", ...).Name("UserSettings")
f.Get("/users/{name}", ...).Name("UsersName")

f.Get(..., func(c flamego.Context) {
   c.URLPath("UserSettings")                         // => /user
   c.URLPath("UserSettings", "withOptional", "true") // => /user/settings
   c.URLPath("UsersName", "name", "joe")             // => /users/joe
})
```

## 自定义 `NotFound` 处理器

默认情况下，[`http.NotFound`](https://pkg.go.dev/net/http#NotFound) 函数会被用于响应 404 状态码的页面，但可以通过 `NotFound` 方法进行自定义：

```go:no-line-numbers
f.NotFound(func() string {
    return "This is a cool 404 page"
})
```

## 自动注册 `HEAD` 方法

默认情况下，使用 `Get` 方法注册的路由只会接受 HTTP GET 方法的请求，但部分 Web 应用可能会希望同时支持 HEAD 请求。

调用 `AutoHead` 方法可以在注册任意路由的 GET 方法的处理器时，自动为该路由的 HEAD 方法也注册相同的处理器：

```go:no-line-numbers
f.Get("/without-head", ...)
f.AutoHead(true)
f.Get("/with-head", ...)
```

需要注意的是，该行为仅会在调用 `AutoHead(true)` 之后的路由配置生效，并不会影响已经配置好的路由。

如上例中，`/with-head` 路径同时接受 GET 和 HEAD 请求，而 `/without-head` 路径仅接受 GET 请求。
