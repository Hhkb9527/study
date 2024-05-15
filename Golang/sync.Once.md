本文使用到的 Go 版本为 1.18，源码位置 go1.18/src/sync/once.go

```go
type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 { // 第一次判断，无需加锁，提效
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock() // 加锁
	defer o.m.Unlock()
	if o.done == 0 { // 二次判断；并发环境下，第一次判断后&&加锁前，可能有其他协程已经调用了Do
		defer atomic.StoreUint32(&o.done, 1) // 这里defer保证在f执行完毕后，才更新done的值；避免并发下f未完成Do就返回
		f()
	}
}
```

