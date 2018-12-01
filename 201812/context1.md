## 1.context剖析之使用技巧
### context背景
因为goroutine，go的并发非常方便，但是这也带来了另外一个问题，当我们进行一个耗时的异步操作时，如何在约定的时间内终止该操作并返回一个自定义的结果？这也是大家常说的我们如何去终止一个goroutine(因为goroutine不同于os线程，没有主动interrupt机制)，这里就轮到今天的主角context登场了。

context源于google，于1.7版本加入标准库，按照官方文档的说法，它是一个请求的全局上下文，携带了截止时间、手动取消等信号，并包含一个并发安全的map用于携带数据。context的API比较简单,接下来我会在具体的使用场景中进行介绍。
### 使用场景一: 请求链路传值
一般来说，我们的根context会在请求的入口处构造如下
```
ctx := context.Background()
```
如果拿捏不准是否需要一个全局的context，可以使用下面这个函数构造

```
ctx := context.TODO()
```
**但是不可以为nil**。

传值使用方式如下
```
package main

import (
	"context"
	"fmt"
)

func func1(ctx context.Context) {
	ctx = context.WithValue(ctx, "k1", "v1")
	func2(ctx)
}
func func2(ctx context.Context) {
	fmt.Println(ctx.Value("k1").(string))
}

func main() {
	ctx := context.Background()
	func1(ctx)
}
```
我们在func1通过WithValue(parent Context, key, val interface{}) Context，赋值k1为v1，在其下层函数func2通过ctx.Value(key interface{}) interface{}获取k1的值，比较简单。这里有个疑问，如果我是在func2里赋值，在func1里面能够拿到这个值吗？答案是不能，context只能自上而下携带值，这个是要注意的一点。
### 使用场景二: 取消耗时操作，及时释放资源
可以考虑这样一个问题，如果没有context包，我们如何取消一个耗时操作呢？我这里模拟了两种写法

- 网络交互场景,经常通过SetReadDeadline、SetWriteDeadline、SetDeadline进行超时取消

```

timeout := 10 * time.Second
t = time.Now().Add(timeout)
conn.SetDeadline(t)
```
- 耗时操作场景，通过select模拟
```
package main

import (
	"errors"
	"fmt"
	"time"
)

func func1() error {
	respC := make(chan int)
	// 处理逻辑
	go func() {
		time.Sleep(time.Second * 3)
		respC <- 10
	}()

	// 超时逻辑
	select {
	case r := <-respC:
		fmt.Printf("Resp: %d\n", r)
		return nil
	case <-time.After(time.Second * 2):
		fmt.Println("catch timeout")
		return errors.New("timeout")
	}
}

func main() {
	err := func1()
	fmt.Printf("func1 error: %v\n", err)
}
```

以上两种方式在工程实践中也会经常用到，下面我们来看下如何使用context进行主动取消、超时取消以及存在多个timeout时如何处理
- 主动取消
```
package main

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"time"
)

func func1(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()
	respC := make(chan int)
	// 处理逻辑
	go func() {
		time.Sleep(time.Second * 5)
		respC <- 10
	}()
	// 取消机制
	select {
	case <-ctx.Done():
		fmt.Println("cancel")
		return errors.New("cancel")
	case r := <-respC:
		fmt.Println(r)
		return nil
	}
}

func main() {
	wg := new(sync.WaitGroup)
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go func1(ctx, wg)
	time.Sleep(time.Second * 2)
	// 触发取消
	cancel()
	// 等待goroutine退出
	wg.Wait()
}
```
- 超时取消
```
package main

import (
	"context"
	"fmt"
	"time"
)

func func1(ctx context.Context) {
	hctx, hcancel := context.WithTimeout(ctx, time.Second*4)
	defer hcancel()

	resp := make(chan struct{}, 1)
	// 处理逻辑
	go func() {
		// 处理耗时
		time.Sleep(time.Second * 10)
		resp <- struct{}{}
	}()

	// 超时机制
	select {
	//	case <-ctx.Done():
	//		fmt.Println("ctx timeout")
	//		fmt.Println(ctx.Err())
	case <-hctx.Done():
		fmt.Println("hctx timeout")
		fmt.Println(hctx.Err())
	case v := <-resp:
		fmt.Println("test2 function handle done")
		fmt.Printf("result: %v\n", v)
	}
	fmt.Println("test2 finish")
	return

}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
	defer cancel()
	func1(ctx)
}
```
对于多个超时时间的处理，可以把上述超时取消例子中的注释打开，会观察到，当处理两个ctx时，时间短的会优先触发，这种情况下，如果只判定一个context的Done()也是可以的，但是一定要保证**调用到两个cancel函数**

### 注意事项
- context只能自顶向下传值，反之则不可以。
- 如果有cancel，一定要保证调用，否则会造成资源泄露，比如timer泄露。
- context一定不能为nil，如果不确定，可以使用context.TODO()生成一个empty的context。

以上是context剖析的上篇，主要从使用层面，让大家有一个直观的认识，这样在工程中可以进行灵活的使用，接下来会从源码层面进行剖析。

### 参考资料
[golang官方包](https://golang.org/pkg/context)

[Go Concurrency Patterns: Context](https://blog.golang.org/context)

[etcd客户端超时处理示例代码](https://github.com/etcd-io/etcd/blob/master/client/client.go#L543)
