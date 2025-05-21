# 4. Go特有概念

## Goroutines与并发模型

Go语言的并发模型是它最引人注目的特性之一，尤其对Java开发者而言。Go采用CSP（通信顺序进程）并发模型，通过轻量级的goroutines和channels实现并发，而不是Java中常见的共享内存和锁的方式。

### Goroutines

Goroutines是Go语言的轻量级线程，由Go运行时管理：

```go
// 创建一个goroutine
go func() {
    fmt.Println("在新的goroutine中执行")
}()

// 带参数的goroutine
go calculateSum(10, 20)
```

与Java线程相比的优势：
- 创建成本极低（仅需几KB内存，而Java线程默认需要1MB）
- 可以轻松创建上千甚至上百万个goroutines
- 调度由Go运行时处理，而非操作系统
- 启动速度快，几乎没有额外开销

### 等待组(WaitGroup)

类似于Java的CountDownLatch，用于等待一组goroutine完成：

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)  // 增加计数器
    
    go func(id int) {
        defer wg.Done()  // 完成时减少计数器
        
        fmt.Printf("工作 %d 完成\n", id)
    }(i)
}

wg.Wait()  // 阻塞直到计数器为0
fmt.Println("所有工作完成")
```

### 互斥锁(Mutex)

与Java类似，Go也提供互斥锁用于共享资源访问控制：

```go
var (
    counter int
    mutex   sync.Mutex
)

func incrementCounter() {
    mutex.Lock()
    defer mutex.Unlock()
    counter++
}
```

也可以使用读写锁（RWMutex）提高并发读取效率：

```go
var (
    data      map[string]string
    rwMutex   sync.RWMutex
)

// 写操作
func updateData(key, value string) {
    rwMutex.Lock()
    defer rwMutex.Unlock()
    data[key] = value
}

// 读操作
func readData(key string) string {
    rwMutex.RLock()
    defer rwMutex.RUnlock()
    return data[key]
}
```

## Channels通信

Channel是Go中goroutine之间通信的管道，实现了"通过通信共享内存"的CSP原则，而非Java中"通过共享内存通信"的方式。

### 基本使用

```go
// 创建无缓冲channel
ch := make(chan int)

// 向channel发送数据（会阻塞直到有人接收）
go func() {
    ch <- 42
}()

// 从channel接收数据（会阻塞直到有数据）
value := <-ch
fmt.Println(value)
```

### 缓冲通道

```go
// 创建缓冲channel，容量为5
bufferedCh := make(chan string, 5)

// 可以发送5个值而不阻塞
bufferedCh <- "消息1"
bufferedCh <- "消息2"

// 读取值
msg := <-bufferedCh
```

### 通道方向

可以限制通道的方向，增加类型安全性：

```go
// 只发送通道
func sendData(ch chan<- int) {
    ch <- 42
    // <-ch  // 编译错误，不能从只发送通道接收
}

// 只接收通道
func receiveData(ch <-chan int) {
    value := <-ch
    // ch <- 10  // 编译错误，不能向只接收通道发送
}
```

### Select语句

类似于Java的NIO Selector，但更简洁和易用：

```go
select {
case msg1 := <-channel1:
    fmt.Println("从channel1收到:", msg1)
    
case msg2 := <-channel2:
    fmt.Println("从channel2收到:", msg2)
    
case channel3 <- "hello":
    fmt.Println("发送到channel3")
    
case <-time.After(500 * time.Millisecond):
    fmt.Println("超时")
    
default:
    fmt.Println("没有channel就绪")
}
```

### 通道关闭和遍历

```go
ch := make(chan int, 5)

// 发送方
go func() {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)  // 关闭通道，表示不再发送
}()

// 接收方
// 方法1：使用for-range循环（直到通道关闭）
for num := range ch {
    fmt.Println(num)
}

// 方法2：检测通道是否关闭
for {
    num, open := <-ch
    if !open {
        break  // 通道已关闭
    }
    fmt.Println(num)
}
```

### 实际应用示例

一个典型的并发处理模式（类似于Java中的生产者-消费者）：

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("工作者 %d 开始处理任务 %d\n", id, job)
        time.Sleep(time.Second) // 模拟处理时间
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // 启动3个工作者
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    
    // 发送5个任务
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    
    // 收集所有结果
    for a := 1; a <= 5; a++ {
        <-results
    }
}
```

## 接口与鸭子类型

Go的接口实现是隐式的，称为"鸭子类型"（duck typing）。如果一个类型实现了接口的所有方法，即被视为实现了该接口，不需要Java中的显式`implements`关键字。

### 接口定义

```go
// 定义接口
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}
```

### 接口实现

任何实现了接口方法的类型都自动满足该接口：

```go
// 文件类型
type File struct {
    // ...
}

// 实现Reader接口（隐式）
func (f *File) Read(p []byte) (n int, err error) {
    // 实现读取逻辑
    return len(p), nil
}

// 实现Writer接口（隐式）
func (f *File) Write(p []byte) (n int, err error) {
    // 实现写入逻辑
    return len(p), nil
}

// File现在自动满足Reader、Writer和ReadWriter接口
```

### 空接口

空接口`interface{}`（Go 1.18后可用`any`代替）类似于Java的`Object`，可接收任何类型：

```go
// 接受任何类型
func printAny(v interface{}) {
    fmt.Println(v)
}

// Go 1.18+
func printAny(v any) {
    fmt.Println(v)
}
```

### 类型断言

类似于Java的类型转换和`instanceof`：

```go
func process(i interface{}) {
    // 方式1：类型断言
    str, ok := i.(string)
    if ok {
        fmt.Println("字符串值:", str)
    }
    
    // 方式2：类型选择（switch）
    switch v := i.(type) {
    case string:
        fmt.Println("字符串:", v)
    case int:
        fmt.Println("整数:", v)
    case []byte:
        fmt.Println("字节切片长度:", len(v))
    default:
        fmt.Println("未知类型")
    }
}
```

### 常见接口模式

```go
// 排序接口
type Sortable interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

// 字符串格式化
type Stringer interface {
    String() string  // 类似Java的toString()
}

// 自定义错误
type CustomError struct {
    message string
}

func (e *CustomError) Error() string {
    return e.message
}
```

## 错误处理机制

Go采用显式错误处理而非Java的异常机制。在Go中，错误是普通的值，通常作为函数的最后一个返回值。

### 基本错误处理

```go
// 返回错误的函数
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("除数不能为0")
    }
    return a / b, nil
}

// 调用并处理错误
result, err := divide(10, 0)
if err != nil {
    fmt.Println("出错:", err)
    return
}
fmt.Println("结果:", result)
```

### 自定义错误

```go
// 方法1：使用errors.New
err := errors.New("发生错误")

// 方法2：使用fmt.Errorf（支持格式化）
err := fmt.Errorf("处理%s时出错", filename)

// 方法3：自定义错误类型
type QueryError struct {
    Query   string
    Message string
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("查询'%s'失败: %s", e.Query, e.Message)
}

// 使用自定义错误
return &QueryError{
    Query:   "SELECT * FROM users",
    Message: "数据库连接失败",
}
```

### 错误封装（Go 1.13+）

```go
// 包装错误
originalErr := errors.New("原始错误")
wrappedErr := fmt.Errorf("上下文信息: %w", originalErr)

// 解包错误
fmt.Println(errors.Unwrap(wrappedErr))

// 错误比较
if errors.Is(wrappedErr, originalErr) {
    fmt.Println("是同一个错误")
}

// 错误类型转换
var queryErr *QueryError
if errors.As(err, &queryErr) {
    fmt.Println("是查询错误:", queryErr.Query)
}
```

### 与Java异常的对比

| Go错误处理 | Java异常处理 |
|------------|-------------|
| 显式，通过返回值 | 隐式，通过抛出异常 |
| 强制检查错误 | 可以忽略异常（未检查异常） |
| 简单的值 | 复杂的对象和堆栈跟踪 |
| 本地化处理 | 可以远距离传播 |
| 无性能开销 | 有性能开销（特别是创建堆栈跟踪） |

### panic和recover

虽然Go鼓励使用错误返回值，但它也提供了panic/recover机制用于处理真正的异常情况（类似Java的try-catch）：

```go
func riskyFunction() {
    // 设置recover处理
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("从panic恢复:", r)
            // 可以记录日志或执行清理操作
        }
    }()
    
    // 触发panic
    panic("发生严重错误")
}
```

最佳实践是：
- 对于可恢复的情况使用错误值
- 仅在真正不可恢复的情况下使用panic
- 在包边界处使用recover转换panic为错误

### 优雅的错误处理模式

处理多个可能出错的操作：

```go
// 不太优雅的方式
file, err := os.Open(filename)
if err != nil {
    return err
}
defer file.Close()

data, err := ioutil.ReadAll(file)
if err != nil {
    return err
}

err = json.Unmarshal(data, &result)
if err != nil {
    return err
}

// 更优雅的方式（使用匿名函数）
var result MyStruct
err := func() error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("打开文件失败: %w", err)
    }
    defer file.Close()
    
    data, err := ioutil.ReadAll(file)
    if err != nil {
        return fmt.Errorf("读取文件失败: %w", err)
    }
    
    if err := json.Unmarshal(data, &result); err != nil {
        return fmt.Errorf("解析JSON失败: %w", err)
    }
    
    return nil
}()

if err != nil {
    // 处理错误
    log.Fatal(err)
}
``` 