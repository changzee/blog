# golang web项目统一响应风格及错误处理机制

这段时间一直在和golang和部门内部开发的go web框架打交道，发现其存在一些些小瑕疵，使用起来跟Gin差距比较明显。比如，由于它缺失了内部缺失了http api返回数据结构规范化及自定义的一些机制，同时对错误处理的方式没有一个比较好用的推荐姿势。所以我在项目应用层写了一两个package来做对应的优化。

### 更加优雅的输出姿势

#### 统一输出结构:

```go
// Standard means standard output struct
type Standard struct {
    Code    int         `json:"code"`              // 业务状态码
    Message string      `json:"message,omitempty"` // 业务信息
    Data    interface{} `json:"data,omitempty"`    // 数据
}
```

另外, 特殊输出格式可以采用与Gin一致的H类型:

```go
type H map[string]interface{}
```

#### 生成对应结构体可以使用以下全局方法:

```go
// JSON 输出规范结构体
func JSON(ctx *web.Context, httpCode int, data *Standard) {
    if response, err := jsoniter.MarshalToString(data); err != nil {
        logger.Errorf("json解析失败: %+v", err)
    } else {
        ctx.Response(httpCode, response)
    }
}

// CustomizeJSON 自定义输出json
func CustomizeJSON(ctx *web.Context, httpCode int, h H) {
    if response, err := jsoniter.ConfigFastest.MarshalToString(h); err != nil {
        logger.Errorf("json解析失败: %+v", err)
    } else {
        response.Response(httpCode, response)
    }
}

// Success 成功输出
func Success(ctx *web.Context, data ...interface{}) {
    JSON(http.StatusOK, &Standard{Data: data})
}

// String 直接输出
func String(ctx *web.Context, httpCode int, data ...string) {
    message := ""
    if len(data) > 0 {
        message = data[0]
    }

    ctx.Response(httpCode, message)
}
```

#### 使用姿势:

```go
// 统一返回格式
func Echo(ctx *web.COntext) {
    response.JSON(ctx, http.StatusOK, &response.Standard{
        "words": "Hello world"
    })
}

// 自定义返回
// Metrics 返回运行状态信息
func Metrics(ctx *web.Context) {
    memStat := runtime.MemStats{}
    runtime.ReadMemStats(&memStat)
    response.CustomizeJSON(ctx, http.StatusOK, response.H{
        "NumCPU":       runtime.NumCPU(),
        "NumGoroutine": runtime.NumGoroutine(),
        "QPS":          middleware.ReqQps.GetQps(),
        "memStat":      memStat,
    })
}
```

### 统一错误处理机制

“大道至简”的Go采用的错误处理机制一直饱受争议，其设计理念可以参考社区的博客(https://blog.golang.org/errors-are-values)。这种机制给很多人的感觉像是回退到了C语言，处理起来对比(try-catch/monad)来说比较丑陋麻烦, 而且err也容易写飞(很多新人直接`return errors.New("这里发生了个小错误")`)。接下来将探讨如何在这种环境下夹缝求生。

#### 针对WEB请求错误而定义的错误类型

为了与上面的输出结构统一，定义了以下的错误结构体:

```go
// Error 统一错误处理结构
type Error struct {
    HttpCode int    `json:"http_status"` // 对应http状态码
    Code     int    `json:"code"`        // 错误返回码
    Message  string `json:"message"`     // 错误信息
}

// Error 获取错误信息
func (e Error) Error() string {
    return e.Message
}

// GetErrCode 获取业务错误码
func (e Error) GetErrCode() int {
    return e.Code
}

// GetHttpCode 获取http码
func (e Error) GetHttpCode() int {
    return e.HttpCode
}

// New 新建错误
func New(code int, message string, httpCode int) Error {
    return Error{httpCode, code, message}
}
```

同时，在response层新增了一个方法,用于统一返回错误信息:

 ```go
// Error 返回错误结构体，后面有详细的实现
func Error(ctx *web.Context, err error) {
    var target errors.Error
    JSON(target.GetHttpCode(), &Standard{target.GetErrCode(), target.Error(), })
}
 ```

这样，在控制器层，一旦接收到err我们想返回的话，直接:

```go
if err != nil {
	 response.Error(err)
}
```

#### 错误值统一定义

错误值可以是满足语言定义的error 接口的任何类型。程序可以使用类型断言(type assertion)或类型开关(type switch)来判断错误值是否可被视为特定的错误类型。但是这样的话你需要为每种错误定义对应的类型，不好做处理。如:

```go
type NotFoundError struct {
    Name string
}

func (e *NotFoundError) Error() string { return e.Name + ": not found" }

if e, ok := err.(*NotFoundError); ok {
    // e.Name wasn't found
}
```

直接定义对应全局错误，然后直接通过比较来判断是否发生对应的错误会更加方便些:

```go
var ErrNotFound = errors.New("not found")

if err == ErrNotFound {
    // something wasn't found
}
```

所以我在应用层定义了一个global文件，用于定义全局错误，同时也方便查看还有生成文档:

```go
var (
    ValidationError   = New(400, "输入参数有误", http.StatusBadRequest)
    ForbiddenError    = New(403, "用户未登录", http.StatusForbidden)
    NotFoundError     = New(404, "找不到请求的资源", http.StatusNotFound)

    VirtualNoLoginError             = New(110000, "请先登陆哦", http.StatusOK)
    OrderNotExistError              = New(120000, "订单不存在", http.StatusOK)
    OrderSkuNotExistError           = New(120002, "订单Sku不存在", http.StatusOK)
    OrderConfirmReceiptError        = New(120003, "当前订单中有退货中的商品，完成退款后才能确认收货哦~", http.StatusOK)
    CartPublishSkuExistError        = New(124000, "购物车中已经有相同商品，无法修改规格哦~", http.StatusOK)
)
```

这样的话，如果说上层想知道下层发生了什么错误可以直接做比较:

```go
if err == ValidationError {
	// logger.Error(“验证错误: %+v”, err)
} else if err == OrderNotExistError {
	// 一些业务处理
}
```

#### 接口返回错误

如何返回错误在go中是一个比较麻烦的事情，特别要区分是业务错误和系统错误。我之前的想法是函数返回两个err, 一个用于标志业务错误，另一个用于标志系统错误, 后面想想这种方式有些丑陋。后来基于标准库1.13版本改写了下`response.Error`方法，统一了两者:

```go
// Error 返回错误结构体...
func Error(err error) {
    var target errors.Error

    // 如果错误为业务错误类型则进行特殊格式化处理
    if errors.As(err, &target) {        
        JSON(target.GetHttpCode(), &Standard{target.GetErrCode(), target.Error(), })
    } else {
        logger.Errorf("响应异常: %+v", err)
        
        // 测试环境与本地开发环境直接打印错误，方便debug
        if Env == DEV || Env == TEST {
            String(http.StatusInternalServerError, fmt.Sprintf("系统错误: %+v", err))
        } else {
            String(http.StatusInternalServerError, http.StatusText(http.StatusInternalServerError))
        }
    }
}

```

同时对系统错误做了下特殊处理，在测试与开发环境下抛出对应的调用栈，只需要在抛出错误的时候使用:

```go
import errors2 "github.com/pkg/errors"

if err != nil {
    // 为错误包一层调用栈信息，这样response.Error能够直接打印
	return errors2.WithStack(errors.ServiceInnerError)
}
```



#### 携带数据的错误值

如何为错误值包裹一层数据是件比较难的事，因为如果直接修改全局错误的话会导致在上层不能比对特定的错误，对此借鉴了标准库的一些写法:

```go
// withData is a error with some specific data.
type withData struct {
    error
    data interface{}
}

// Data function gets the Error data of withData
func (w *withData) Data() interface{} {
    return w.data
}

// WrapData function wraps error with data.
func WithData(err error, data interface{}) error {
    if err == nil {
        return nil
    }
    return &withData{err, data}
}

// Unwrap provides compatibility for Go 1.13 error chains.
func (w *withData) Unwrap() error { return w.error }
```

这样想返回错误数据的话只需要:

```go
if err != nil {
	return errors.WithData(errors.OrderNotExistError, map[string]interface{}{
		"order_id": orderId // 顺便返回对应的订单ID
	})
}
```

最后需要修改下`response.Error`方法:

```go
// Error 返回错误结构体.
func Error(err error) meetyou.HttpCodeResponse {
    var target errors.Error

    // 如果错误为业务错误类型则进行特殊格式化处理
    if errors.As(err, &target) {
        var errData interface{} = nil
        if tmp, ok := err.(interface{Data() interface{}}); ok {
            errData = tmp.Data()
        }
        return JSON(target.GetHttpCode(), &Standard{target.GetErrCode(), target.Error(), errData})
    } else {
        logger.Errorf("响应异常: %+v", err)
        if meetyou.Env == meetyou.DEV || meetyou.Env == meetyou.TEST {
            return String(http.StatusInternalServerError, fmt.Sprintf("系统错误: %+v", err))
        } else {
            return String(http.StatusInternalServerError, http.StatusText(http.StatusInternalServerError))
        }
    }
}

```

### 其他

#### 发生系统错误的怎么办:

1. 对于异常, 直接panic，应用层使用recover恢复，方便排查错误

2. 日志记录错误，非业务错误需层层进行记录，方便排查问题

   ```go
   import errors2 "github.com/pkg/errors"
   
   if err != nil {
        err = errors2.WithStack(err)
   	logger.Errorf("xxx发生错误: %+v", err)
        // 为错误包一层调用栈信息，这样response.Error能够直接打印
   	return err
   }
   ```
