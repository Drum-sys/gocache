# Cocurrency-in-Go
## Cocurrency Patterns
### for-select loop
```go
	done := make(chan interface{})
	stringStream := make(chan string, 5)
  // 在chan上发送迭代变量
	for _, s := range []string{"a", "b", "c"}{
		select {
		case <-done:
			return
		case stringStream <- s:
		}
	}
  // 循环等待退出条件
	for {
		select {
		case <-done:
			return
		default:
			//do something
		}
	}
```

### Preventing Goroutine Leaks
父goroutine应该告诉子goroutine什么时候退出，一般情况下传递一个无缓冲的channel，结合for select使用，当关闭无缓冲channel时，select接收到关闭信号，return。避免goroutine一直留在内存中
```go
func main() {
	doWork := func(done <- chan interface{}, strings <-chan string) <-chan interface{} {
		terminated := make(chan interface{})
		go func() {
			defer fmt.Println("dowork done")
			defer close(terminated)
			for {
				select {
				case s := <-strings:
					fmt.Println(s)
				case <-done:
					return
				}
			}
		}()
		return terminated
	}

	done := make(chan interface{})
	terminated := doWork(done, nil)
	go func() {
		time.Sleep(time.Second)
		fmt.Println("Canceling dowork goroutine")
		close(done)
	}()
	<-terminated
	fmt.Println("Done.")
}
```

### Or-channel
一般将多个已完成channel组合为一个已完成channel，当其中一个组件通道关闭，该通道就会关闭
```go
func main() {
	var or func(channels ...<-chan interface{}) <-chan interface{}

	or = func(channels ...<-chan interface{}) <- chan interface{} {
		fmt.Printf("channel length %v\n", len(channels))
		switch len(channels) {
		case 0:
			//fmt.Println("第一次静茹case0")
			return nil
		case 1:
			//fmt.Println("第一次静茹case0")
			return channels[0]
		}
		orDone := make(chan interface{})
		go func() {
			defer close(orDone)
			switch len(channels) {
			case 2:
				fmt.Println("长度等于2")
				select {
				case <-channels[0]:
					fmt.Println("第一次静茹case0")
				case <-channels[1]:
				}
			default:
				select {
				case <-channels[0]:
					fmt.Println("第一次静茹case0")
				case <-channels[1]:
					fmt.Println("第一次静茹case1")
				case <-channels[2]:
					fmt.Println("第一次静茹case2")
				case <-or(append(channels[3:], orDone)...):
				}
			}
		}()
		fmt.Println(">>>>>>>>>>>>>>>>>>")
		return orDone
	}

	sig := func(after time.Duration) <-chan interface{} {
		c := make(chan interface{})
		go func() {
			defer close(c)
			time.Sleep(after)
		}()
		return c
	}

	start := time.Now()
	<-or(
		sig(1*time.Second),
		sig(1*time.Second),
		//sig(2*time.Hour),
		sig(5*time.Millisecond),
		sig(1*time.Second),
		//sig(1*time.Hour),
		//sig(1*time.Minute),
	)
	fmt.Printf("done after %v", time.Since(start))

	time.Sleep(3 * time.Second)

	arr := []int{1,2}
	fmt.Println(append(arr, 3))

}
```

### Error handling
当goroutine当中错误太多时， 应该将错误封装保存起来发送给程序的另一部分，该部分拥有程序状态的完整信息。
func main() {
	checkStatus := func(done <- chan interface{}, urls ...string) <-chan Result {
		results := make(chan Result)
		go func() {
			defer close(results)
			for _, url := range urls{
				var result Result
				rsp, err := http.Get(url)
				//if err != nil {
				//	fmt.Println(err)
				//}
				result = Result{
					Error: err,
					Response: rsp,
				}
				select {
				case <-done:
					return
				case results <-result:
				}
			}
		}()
		return results
	}

	done := make(chan interface{})
	defer close(done)

	urls := []string{"https://www.google.com", "https://badhost"}
	for result := range checkStatus(done, urls...){
		if result.Error != nil {
			fmt.Printf("error : %v", result.Error)
		}
		fmt.Printf("Response: %v\n", result.Response.Status)
	}
}


### Pipelines


