为了创建连接点，您必须同步main goroutine和sayHello goroutine。这可以通过多种方式实现，但我将使用我们将在47页的“sync包”中讨论的一种方式:sync. WaitGroup。现在你不必要理解这个示例如何创建一个连接点，重要的是知道它在两个goroutine之间创建了一个连接点。这个示例的正确版本如下:
```go
var wg sync.WaitGroup
sayHello := func() {
    defer wg.Done()
    fmt.Println("hello")
}
wg.Add(1)
go sayHello()
wg.Wait() ①
```
① 这里是连接点
输出：
```bash
hello
```
这个示例会准确地阻塞main goroutine，直到承载sayHello函数的goroutine终止。您将在第47页的“sync包”中了解sync.WaitGroup的工作原理，但为了使我们的示例正确，我会开始用它来创建连接点。

我们在示例中使用了大量匿名函数来快速创建goroutine示例。让我们把注意力转移到closure（闭包）上。闭包根据其创建的词法作用域捕获变量。如果你在goroutine中运行一个闭包，闭包操作的是这些变量的拷贝？还是原始引用？让我们试试看：
```go
var wg sync.WaitGroup
salutation := "hello"
wg.Add(1)
go func() {
    defer wg.Done()
    salutation = "welcome" ①
}()
wg.Wait()
fmt.Println(salutation)
```
① 这里我们看到goroutine修改了变量salutation的值。
你认为salutation的值是"hello"还是"welcome"?让我们运行一下:
```bash
welcome
```
这证明了goroutine在创建它们的地址空间中执行，因此我们的程序打印出"welcome"一词。你认为这个程序会输出什么？
```go
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(salutation) ①
    }()
}
wg.Wait()
```
① 这里我们测试打印字符串切片创建的循环变量salutation。
答案超出大多数人的预想，这是Go中为数不多的令人惊讶的事情之一。大多数人直觉上认为这会以某种不确定的顺序打印出"hello"、"welcome"和"good day"，但看看它的结果：
```bash
good day
good day
good day
```
这有点令人惊讶!我们来看看这是怎么回事。在本例中，goroutine正在运行一个含有循环变量salutation的闭包，salutation的类型为string。在循环中，切片中下一个字符串的值赋值给salutation。由于被调度的goroutine可能在未来的任何时间点运行，因此无法确定将从goroutine中打印哪些值。在我的机器上，循环很有可能在goroutine开始之前就退出了。这意味着变量salutation超出了作用域。然后会发生什么?goroutine仍然可以引用超出作用域的内容吗?goroutine会不会访问可能已被垃圾收集的内存?

这从侧面说明了Go如何管理内存。Go的运行时会持续监测，知道仍然存在对salutation变量的引用，因此会将内存转移到堆中，以便goroutine可以继续访问它。
通常在我的机器上，循环在任何goroutine开始运行之前退出，因此salutation被转移到包含对字符串切片中最后一个值——"good day"的引用的堆中.所以会看到“good day”打印了三次。这个循环的正确写法是将一个salutation的拷贝传递到闭包中，这样当goroutine运行时，salutation将对循环中的数据进行操作:
```go
var wg sync.WaitGroup
for _, salutation := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(salutation string) { ①
        defer wg.Done()
        fmt.Println(salutation)
    }(salutation) ②
}
wg.Wait()
```
① 这里我们像其他函数一样声明一个形参。我们将原始的salutation变量映射到更加明显的位置。
② 这里我们将当前循环的变量传递给闭包。生成字符串的拷贝，从而确保在运行goroutine时引用正确的字符串。
我们看到正确的输出了：
```bash
good day 
hello 
greetings
```
这个例子的行为和我们期望的一样，只是稍微有点冗长
因为goroutine彼此在相同的地址空间中操作，并且只是承载函数，所以利用goroutine只是是编写非并发代码的自然扩展。Go的编译器很好地处理了内存中的固定变量，因此goroutine不会意外地访问释放的内存，这使得开发人员可以专注于他们的问题而不是内存管理;然而，这不是一张空白支票。

由于多个goroutine可以对同一个地址空间进行操作，所以我们仍然需要担心同步问题。正如我们已经讨论过的，我们可以选择同步访问共享内存，或者我们可以使用CSP原语通过通信共享内存。我们将在本章第64页的“channel”和第47页的“sync包”中讨论这些技术。

goroutine的另一个好处是非常轻量。

摘自Go FAQ：
>一个新创建的goroutine被赋予了几千字节，这几乎总是足够的。
如果不是，则运行时自动增加(或缩小)用于存储堆栈的内存，允许许多goroutine驻留在适量的内存中。每次函数调用的CPU开销平均约为3条廉价指令。在同一个地址空间中创建数十万个goroutine是可行的。如果goroutine只是线程，那么系统资源将以更小的数量耗尽。

每个goroutine几千字节;这一点也不坏!让我们自己来验证一下。但在此之前，我们必须介绍关于goroutines的一件有趣的事情:垃圾收集器不会收集以某种方式被丢弃的goroutines。如果我这样写:
```go
go func() {
    // <operation that will block forever>
}()
// Do work
```
这里的goroutine将一直存在，直到进程退出。我们将在第4章第90页的“防止Goroutine泄漏”部分讨论如何解决这个问题。

在下一个示例中，我们将利用这一点来实际测量goroutine的大小。

在下面的例子中，我们结合了这样一个事实，即goroutine不会被垃圾收集，运行时能够自省，并测量goroutine创建前后分配的内存量:
```go
memConsumed := func() uint64 {
    runtime.GC()
    var s runtime.MemStats
    runtime.ReadMemStats(&s)
    return s.Sys
}
var c <-chan interface{}
var wg sync.WaitGroup
noop := func() { wg.Done(); <-c } ①
const numGoroutines = 1e4 ②
wg.Add(numGoroutines)
before := memConsumed() ③
for i := numGoroutines; i > 0; i-- {
    go noop()
}
wg.Wait()
after := memConsumed() ④
fmt.Printf("%.3fkb", float64(after-before)/numGoroutines/1000)
```
① 我们需要一个永远不会退出的goroutine，这样我们就可以在内存中保存一些goroutine来进行测量。现在不用担心我们是怎么做到的;只要知道这个goroutine直到进程完成才会退出。
② 这里我们定义要创建的goroutine的数量。我们将使用大数定律来渐近地接近一个goroutine的大小。
③ 在这里，我们在创建goroutine之前测量内存消耗量。
④ 在这里，我们测量创建goroutine后所消耗的内存量
结果如下
```
2.817kb
```
看起来文档是正确的!这些只是空的goroutine，不做任何事情，但它仍然让我们知道我们可能创建的goroutine的数量。表3-1给出了在不使用交换空间的情况下使用64位CPU可能创建多少个goroutines的粗略估计。

这些数字相当大!在我的笔记本电脑上，我有8gb的内存，这意味着理论上我可以在不需要交换的情况下启动数百万个goroutine。当然，这忽略了在我的计算机上运行的其他东西，以及goroutines的实际内容，但是这个快速计算证明了goroutines是多么轻量级!

上下文切换可能会让我们感到沮丧，这是指承载一个并发进程的某些东西必须保存其状态以切换到运行另一个并发进程。如果我们有太多的并发进程，我们可能会花费所有的CPU时间在它们之间进行上下文切换，并且永远不会完成任何实际工作。在操作系统级别，对于线程，这可能代价相当大。操作系统线程必须保存寄存器值、查找表和内存映射等内容，以便在适当的时候成功地切换回当前线程。然后它必须为传入线程加载相同的信息。

软件中的上下文切换相对来说要便宜得多。在软件定义的调度器下，运行时可以在持久化检索内容、持久化方式以及持久化需要发生的时间方面更有选择性。让我们来看看在我的笔记本电脑上，在OS线程和goroutine之间进行上下文切换的相对性能。首先，我们将利用Linux内置的基准测试套件来测量在同一个内核上的两个线程之间发送消息所需的时间:
```bash
taskset -c 0 perf bench sched pipe -T
```
输出
```bash
# Running 'sched/pipe' benchmark:
# Executed 1000000 pipe operations between two threads
     Total time: 2.935 [sec]
       2.935784 usecs/op
         340624 ops/sec
```
这个基准测试实际上测量了在线程上发送和接收消息所花费的时间，因此我们将得到结果并将其除以2。这样每个上下文文本开关就有1.467 μs。这看起来并不是很糟糕，但是让我们在检查goroutines之间的上下文切换之前保留判断。

我们将使用Go构建一个类似的基准测试。我使用了一些我们还没有讨论过的东西，所以如果有任何困惑，只要遵循标注并关注结果。

下面的例子将创建两个goroutine并在它们之间发送消息:
```go
func BenchmarkContextSwitch(b *testing.B) {
    var wg sync.WaitGroup
    begin := make(chan struct{})
    c := make(chan struct{})
    var token struct{}
    sender := func() {
        defer wg.Done()
        <-begin 
        for i := 0; i < b.N; i++ {
            c <- token 
        }
    }
    receiver := func() {
        defer wg.Done()
        <-begin 
        for i := 0; i < b.N; i++ {
            <-c 
        }
    }
    wg.Add(2)
    go sender()
    go receiver()
    b.StartTimer() 
    close(begin) 
    wg.Wait()
}
```
在这里，我们等着被告知开始。我们不希望设置和启动每个goroutine的成本影响到上下文切换的度量。

这里我们向接收方发送消息。struct{}{}被称为空结构体，不占用内存;因此，我们只测量发送一条消息所需的时间。

在这里，我们接收到一条消息，但对它什么也不做。

这里我们开始性能计时器。

在这里，我们告诉两个goroutines开始。
我们运行基准测试，指定我们只想利用一个CPU，这样它就与Linux基准测试类似。让我们来看看结果:
```
go test -bench=. -cpu=1 \
src/gos-concurrency-building-blocks/goroutines/fig-ctx-switch_test.go
BenchmarkContextSwitch 5000000 225 ns/op
PASS
ok command-line-arguments 1.393s
```
哇!这是0.225 μs，或者比我机器上的操作系统上下文切换快92%，如果你还记得的话，它需要1.467 μs。很难说有多少个goroutine会导致过多的上下文切换，但我们可以轻松地说，上限不太可能成为使用goroutine的任何障碍。

阅读了本节，您现在应该了解如何开始goroutine以及它们的工作原理。您还应该相信，只要您觉得问题空间需要，就可以安全地创建一个goroutine.你创建的goroutine越多，如果你的问题空间不受阿姆达尔定律所规定的一个并发段的限制，那么你的程序就越能扩展到多个处理器。创建goroutine的成本非常低，因此只有在证明它们是性能问题的根本原因时，才应该讨论它们的成本。

