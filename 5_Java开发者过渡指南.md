# 5. Java开发者过渡指南

对资深Java开发者来说，迁移到Go语言时会遇到一些概念和设计理念的差异。本章将重点讨论这些差异，帮助Java开发者更顺利地过渡到Go语言开发。

## 面向对象vs组合设计

### Java的类继承 vs Go的组合

Java是一种纯粹的面向对象语言，而Go采用的是更为简单的组合设计：

```go
// Go中没有类和继承，而是使用结构体和组合
type Animal struct {
    Name string
}

func (a *Animal) Speak() string {
    return "某种声音"
}

// 组合而非继承
type Dog struct {
    Animal  // 嵌入Animal结构体
    Breed string
}

// 可以重写方法
func (d *Dog) Speak() string {
    return "汪汪"
}
```

对比Java的实现：

```java
// Java实现
class Animal {
    private String name;
    
    public String speak() {
        return "某种声音";
    }
}

class Dog extends Animal {
    private String breed;
    
    @Override
    public String speak() {
        return "汪汪";
    }
}
```

### 转变思维方式

1. **从继承思考转向组合思考**：
   - Java: "Dog是一种Animal"（is-a关系）
   - Go: "Dog中包含Animal的特性"（has-a关系）

2. **从类思考转向接口思考**：
   - Java中先定义具体类，再抽象接口
   - Go中先定义小接口，再实现具体类型

3. **代码示例**：构建一个简单的日志系统

```go
// Go风格
type Logger interface {
    Log(message string)
}

// 文件日志
type FileLogger struct {
    filePath string
}

func (f *FileLogger) Log(message string) {
    // 写入文件实现
}

// 控制台日志
type ConsoleLogger struct {}

func (c *ConsoleLogger) Log(message string) {
    fmt.Println(message)
}

// 使用
func process(logger Logger) {
    logger.Log("处理中...")
}
```

vs Java风格：

```java
// Java风格
interface Logger {
    void log(String message);
}

class AbstractLogger implements Logger {
    // 通用实现
}

class FileLogger extends AbstractLogger {
    private String filePath;
    
    @Override
    public void log(String message) {
        // 写入文件实现
    }
}

class ConsoleLogger extends AbstractLogger {
    @Override
    public void log(String message) {
        System.out.println(message);
    }
}
```

### 设计模式的变化

一些Java常用的设计模式在Go中实现方式会有所不同：

1. **单例模式**：Go使用包级变量和sync.Once

```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

2. **策略模式**：Go倾向于使用函数类型

```go
type SortStrategy func([]int) []int

func BubbleSort(data []int) []int {
    // 冒泡排序实现
    return data
}

func QuickSort(data []int) []int {
    // 快速排序实现
    return data
}

// 使用
func Sort(data []int, strategy SortStrategy) []int {
    return strategy(data)
}
```

3. **工厂模式**：Go使用简单函数

```go
func CreateLogger(logType string) Logger {
    switch logType {
    case "file":
        return &FileLogger{filePath: "app.log"}
    case "console":
        return &ConsoleLogger{}
    default:
        return &ConsoleLogger{}
    }
}
```

## 内存管理与GC对比

### 内存分配

Java和Go都使用垃圾回收，但机制有很大不同：

1. **Go的内存分配**:
   - 栈分配优先：Go编译器会尽可能将对象分配在栈上
   - 堆分配更轻量：当对象需要逃逸到堆时，分配开销小
   - 小对象池：小于32KB的对象可能在特殊的小对象池中分配

2. **与Java对比**:
   - Java的对象默认分配在堆上，基本类型和引用在栈上
   - Java的逃逸分析相对不那么积极
   - Go的内存分配通常比Java快

### 垃圾回收器

1. **Go的GC特点**:
   - 并发三色标记清除
   - 低延迟（GC暂停通常<1ms）
   - 非分代（与Java不同）
   - 更简单、更可预测

2. **Java的GC特点**:
   - 多种GC算法：G1、ZGC、Parallel GC等
   - 分代回收（新生代、老年代）
   - 通常需要更多调优

3. **性能对比**:
   - Go在内存使用率和GC暂停时间上通常更有优势
   - Java的GC技术更成熟，提供更多调优选项
   - Go更适合对延迟敏感的应用

### 内存管理最佳实践

1. **Go中避免内存泄漏**:
   ```go
   // 避免在循环中创建闭包捕获大变量
   for _, item := range largeSlice {
       processingFunction(item)  // 而不是 go func(){use(largeSlice)}()
   }
   
   // 及时关闭资源
   f, err := os.Open("file.txt")
   if err != nil {
       return err
   }
   defer f.Close()  // 确保关闭
   ```

2. **与Java对比**:
   - Java需要注意集合类引用、静态引用等
   - Go主要关注goroutine泄漏和延迟释放
   - Go不需要使用弱引用等复杂机制

## 泛型实现差异

Go从1.18版本开始支持泛型，但与Java的泛型有很大不同。

### Go的泛型（1.18+）

基本语法：

```go
// 定义泛型函数
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// 使用泛型函数
minInt := Min[int](10, 20)        // 显式类型参数
minFloat := Min(10.5, 20.5)       // 类型推断

// 泛型类型
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    
    n := len(s.items) - 1
    item := s.items[n]
    s.items = s.items[:n]
    return item, true
}

// 使用泛型类型
intStack := Stack[int]{}
intStack.Push(10)
```

### 与Java泛型对比

1. **类型约束**:
   - Go使用接口作为约束
   - Java使用通配符和bounds

   ```go
   // Go
   func Process[T interface{ String() string }](items []T) {}
   
   // 或使用约束包
   import "golang.org/x/exp/constraints"
   
   func Sort[T constraints.Ordered](items []T) {}
   ```

   ```java
   // Java
   public <T extends Comparable<T>> void sort(List<T> items) {}
   ```

2. **类型擦除**:
   - Java的泛型是编译时特性，运行时会擦除类型信息
   - Go的泛型在运行时保留类型信息
   - Go为每个具体类型实例化一个新函数/类型

3. **限制**:
   - Go的泛型相对简单，某些复杂场景可能受限
   - Java泛型更成熟，支持更多高级用例
   - Go没有Java中的PECS（Producer Extends, Consumer Super）原则

### 实际应用示例

一个通用的泛型仓储模式：

```go
// Go实现
type Repository[T any, ID comparable] interface {
    FindByID(id ID) (T, bool)
    Save(entity T) error
    Delete(id ID) error
}

type InMemoryRepository[T any, ID comparable] struct {
    data map[ID]T
    idExtractor func(T) ID
}

func NewInMemoryRepository[T any, ID comparable](idExtractor func(T) ID) *InMemoryRepository[T, ID] {
    return &InMemoryRepository[T, ID]{
        data: make(map[ID]T),
        idExtractor: idExtractor,
    }
}

func (r *InMemoryRepository[T, ID]) FindByID(id ID) (T, bool) {
    entity, found := r.data[id]
    return entity, found
}

func (r *InMemoryRepository[T, ID]) Save(entity T) error {
    id := r.idExtractor(entity)
    r.data[id] = entity
    return nil
}

// 使用
type User struct {
    ID   int
    Name string
}

userRepo := NewInMemoryRepository[User, int](func(u User) int { return u.ID })
userRepo.Save(User{ID: 1, Name: "张三"})
user, found := userRepo.FindByID(1)
```

## 并发模型转换

Java和Go的并发模型有根本性差异，从Java过渡到Go需要一些思维转变。

### 从线程到Goroutine

Java通常使用线程池管理并发任务：

```java
// Java线程池
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    // 任务实现
});
```

Go使用goroutine更轻量级地处理并发：

```go
// Go goroutines
for i := 0; i < 10000; i++ {
    go func(id int) {
        // 任务实现
    }(i)
}
```

### 从锁到Channel

Java依赖锁和同步块处理共享状态：

```java
// Java同步
class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

Go推荐使用Channel而非锁：

```go
// Go通道
func worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        results <- process(job)
    }
}

// 启动工作池
jobs := make(chan Job, 100)
results := make(chan Result, 100)

for i := 0; i < 3; i++ {
    go worker(i, jobs, results)
}
```

### 从Future/CompletableFuture到Channel

Java使用Future处理异步结果：

```java
// Java Future
Future<String> future = executor.submit(() -> {
    // 长时间操作
    Thread.sleep(1000);
    return "结果";
});

String result = future.get(); // 阻塞等待
```

Go使用Channel实现类似功能：

```go
// Go实现
func fetchData() <-chan string {
    resultCh := make(chan string)
    
    go func() {
        // 长时间操作
        time.Sleep(time.Second)
        resultCh <- "结果"
        close(resultCh)
    }()
    
    return resultCh
}

// 使用
result := <-fetchData() // 阻塞等待
```

### Fan-Out/Fan-In模式

Go让复杂的并发模式实现变得简单：

```go
func fanOut(source <-chan int, n int) []<-chan int {
    channels := make([]<-chan int, n)
    
    for i := 0; i < n; i++ {
        channels[i] = worker(source)
    }
    
    return channels
}

func fanIn(channels []<-chan int) <-chan int {
    result := make(chan int)
    
    for i := range channels {
        ch := channels[i]
        go func() {
            for val := range ch {
                result <- val
            }
        }()
    }
    
    return result
}

func worker(source <-chan int) <-chan int {
    output := make(chan int)
    
    go func() {
        defer close(output)
        for num := range source {
            output <- num * 2
        }
    }()
    
    return output
}
```

## 代码风格与最佳实践

### 代码组织

1. **Java项目结构**:
   ```
   com.example.project/
       controller/
           UserController.java
       service/
           UserService.java
           UserServiceImpl.java
       repository/
           UserRepository.java
       model/
           User.java
       util/
           StringUtils.java
   ```

2. **Go项目结构**:
   ```
   project/
       cmd/
           api/
               main.go
       internal/
           handler/
               user_handler.go
           service/
               user_service.go
           repository/
               user_repository.go
       pkg/
           models/
               user.go
           utils/
               string_utils.go
   ```

### 命名约定

1. **包命名**:
   - Java: `com.company.project.feature`
   - Go: 简短的一个词，如`handler`、`service`

2. **接口命名**:
   - Java: 通常以"I"开头或以"able"、"er"结尾
   - Go: 通常以"er"结尾，如`Reader`、`Writer`

3. **方法命名**:
   - Java: 驼峰式，动词开头，如`getUserById`
   - Go: 驼峰式，但导出(公开)方法首字母大写，如`GetUserByID`

### 错误处理

1. **Java异常处理**:
   ```java
   try {
       User user = userService.findById(id);
       return user;
   } catch (NotFoundException e) {
       throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found");
   } catch (Exception e) {
       logger.error("Error finding user", e);
       throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR);
   }
   ```

2. **Go错误处理**:
   ```go
   user, err := userService.FindByID(id)
   if err != nil {
       if errors.Is(err, ErrNotFound) {
           return nil, status.Error(codes.NotFound, "用户未找到")
       }
       log.Printf("查找用户错误: %v", err)
       return nil, status.Error(codes.Internal, "内部服务错误")
   }
   return user, nil
   ```

### 依赖注入

1. **Java (Spring)**:
   ```java
   @Service
   public class UserServiceImpl implements UserService {
       private final UserRepository userRepository;
       
       @Autowired
       public UserServiceImpl(UserRepository userRepository) {
           this.userRepository = userRepository;
       }
   }
   ```

2. **Go (手动注入)**:
   ```go
   type UserService struct {
       repo UserRepository
   }
   
   func NewUserService(repo UserRepository) *UserService {
       return &UserService{
           repo: repo,
       }
   }
   
   // 在main中
   repo := NewUserRepository(db)
   service := NewUserService(repo)
   handler := NewUserHandler(service)
   ```

### 测试方法

1. **Java (JUnit)**:
   ```java
   @Test
   public void testGetUser() {
       // Given
       when(userRepository.findById(1L)).thenReturn(new User(1L, "张三"));
       
       // When
       User user = userService.getUserById(1L);
       
       // Then
       assertEquals("张三", user.getName());
   }
   ```

2. **Go测试**:
   ```go
   func TestGetUser(t *testing.T) {
       // 准备
       mockRepo := &MockUserRepository{}
       mockRepo.On("FindByID", 1).Return(&User{ID: 1, Name: "张三"}, nil)
       
       service := NewUserService(mockRepo)
       
       // 执行
       user, err := service.GetUserByID(1)
       
       // 验证
       assert.NoError(t, err)
       assert.Equal(t, "张三", user.Name)
   }
   ``` 