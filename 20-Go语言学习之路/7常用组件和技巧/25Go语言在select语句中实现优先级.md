# Go语言在select语句中实现优先级

> https://www.liwenzhou.com/posts/Go/priority_in_go_select/

本文回顾了Go语言中`select`语句的一些用法，并延伸出了一个如何在`select`中实现优先级的小技巧，希望能对大家有所帮助。

# Go语言在select语句中实现优先级

## select语句介绍

Go 语言中的 `select`语句用于监控并选择一组`case`语句执行相应的代码。它看起来类似于`switch`语句，但是`select`语句中所有`case`中的表达式都必须是`channel`的发送或接收操作。一个典型的`select`使用示例如下：

```go
select {
case <-ch1:
	fmt.Println("liwenzhou.com")
case ch2 <- 1:
	fmt.Println("q1mi")
}
```

Go 语言中的 `select` 关键字也能够让当前 `goroutine` 同时等待`ch1` 的可读和`ch2`的可写，在`ch1`和`ch2`状态改变之前，`select` 会一直阻塞下去，直到其中的一个 `channel` 转为就绪状态时执行对应`case`分支的代码。如果多个`channel`同时就绪的话则随机选择一个`case`执行。

除了上面展示的典型示例外，接下来我们逐一介绍一些`select`的特殊示例。

### 空select

空`select`指的是内部不包含任何`case`，例如：

```go
select{
  
}
```

空的 `select` 语句会直接阻塞当前的`goroutine`，使得该`goroutine`进入无法被唤醒的永久休眠状态。

### 只有一个case

如果`select`中只包含一个`case`，那么该`select`就变成了一个阻塞的`channel`读/写操作。

```go
select {
case <-ch1:
	fmt.Println("liwenzhou.com")
}
```

上面的代码，当`ch1`可读时会执行打印操作，否则就会阻塞。

### 有default语句

如果`select`中还可以包含`default`语句，用于当其他`case`都不满足时执行一些默认操作。

```go
select {
case <-ch1:
	fmt.Println("liwenzhou.com")
default:
	time.Sleep(time.Second)
}
```

上面的代码，当`ch1`可读时会执行打印操作，否则就执行`default`语句中的代码，这里就相当于做了一个非阻塞的`channel`读取操作。

### 总结

1. `select` 不存在任何的 `case`：永久阻塞当前 goroutine
2. `select` 只存在一个 `case`：阻塞的发送/接收
3. `select` 存在多个 `case`：随机选择一个满足条件的`case`执行
4. `select` 存在 `default`，其他`case`都不满足时：执行`default`语句中的代码

## 如何在select中实现优先级

已知，当`select` 存在多个 `case`时会随机选择一个满足条件的`case`执行。

现在我们有一个需求：我们有一个函数会持续不间断地从`ch1`和`ch2`中分别接收任务1和任务2，

如何确保当`ch1`和`ch2`同时达到就绪状态时，优先执行任务1，在没有任务1的时候再去执行任务2呢？

高级Go语言程序员小明挠了挠头写出了如下函数：

```go
func worker(ch1, ch2 <-chan int, stopCh chan struct{}) {

	for {
		select {
		case <-stopCh:
			return
		case job1 := <-ch1:
			fmt.Println(job1)
		default:
			select {
			case job2 := <-ch2:
				fmt.Println(job2)
			default:
			}
		}
	}
}
```

上面的代码通过嵌套两个`select`实现了”优先级”，看起来是满足题目要求的。但是这代码有点问题，如果`ch1`和`ch2`都没有达到就绪状态的话，整个程序不会阻塞而是进入了死循环。

怎么办呢？

小明又挠了挠头，又写下了另一个解决方案：

```go
func worker2(ch1, ch2 <-chan int, stopCh chan struct{}) {
	for {
		select {
		case <-stopCh:
			return
		case job1 := <-ch1:
			fmt.Println(job1)
		case job2 := <-ch2:
		priority:
			for {
				select {
				case job1 := <-ch1:
					fmt.Println(job1)
				default:
					break priority
				}
			}
			fmt.Println(job2)
		}
	}
}
```

这一次，小明不仅使用了嵌套的`select`，还组合使用了`for`循环和`LABEL`来实现题目的要求。上面的代码在外层`select`选中执行`job2 := <-ch2`时，进入到内层`select`循环继续尝试执行`job1 := <-ch1`,当`ch1`就绪时就会一直执行，否则跳出内层`select`。

## 实际应用场景

上面的需求虽然是我编的，但是关于在`select`中实现优先级在实际生产中是有实际应用场景的，例如[K8s的controller](https://github.com/kubernetes/kubernetes/blob/7509c4eb478a3ab94ff26be2b4068da53212d538/pkg/controller/nodelifecycle/scheduler/taint_manager.go#L244)中就有关于上面这个技巧的实际使用示例，这里在关于`select`中实现优先级相关代码的关键处都已添加了注释，具体逻辑这里就不展开细说了。

```go
// kubernetes/pkg/controller/nodelifecycle/scheduler/taint_manager.go 
func (tc *NoExecuteTaintManager) worker(worker int, done func(), stopCh <-chan struct{}) {
	defer done()

	// 当处理具体事件的时候，我们会希望 Node 的更新操作优先于 Pod 的更新
	// 因为 NodeUpdates 与 NoExecuteTaintManager无关应该尽快处理
	// -- 我们不希望用户(或系统)等到PodUpdate队列被耗尽后，才开始从受污染的Node中清除pod。
	for {
		select {
		case <-stopCh:
			return
		case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
			tc.handleNodeUpdate(nodeUpdate)
			tc.nodeUpdateQueue.Done(nodeUpdate)
		case podUpdate := <-tc.podUpdateChannels[worker]:
			// 如果我们发现了一个 Pod 需要更新，我么你需要先清空 Node 队列.
		priority:
			for {
				select {
				case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
					tc.handleNodeUpdate(nodeUpdate)
					tc.nodeUpdateQueue.Done(nodeUpdate)
				default:
					break priority
				}
			}
			// 在 Node 队列清空后我们再处理 podUpdate.
			tc.handlePodUpdate(podUpdate)
			tc.podUpdateQueue.Done(podUpdate)
		}
	}
}
```

## 总结

本文回顾了Go语言中`select`语句的一些用法，并延伸出了一个如何在`select`中实现优先级的小技巧，希望能对大家有所帮助。

最后多嘴一句，Go语言由于自身没有很多奇怪的语法糖和自带代码格式化，相比其他语言来说并不会存在看不懂别人写的代码的情况。所以我们完全可以通过阅读优秀库的源代码，与巨人为伍，与高朋为伴，最终吃上更好的饭。