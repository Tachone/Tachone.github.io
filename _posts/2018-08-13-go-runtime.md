---
layout: post
title: "使用 runtime.SetFinalizer 优雅关闭后台 goroutine"
categories: [tech]
tags: [go, runtime]
---

在日常项目开发中，总会使用后台 goroutine 做一些定期清理或更新的任务，这就涉及到 goroutine 生命周期的管理。

## 常规做法：显式 Stop()

对于和主程序生命周期基本一致的后台 goroutine，一般采用如下显式的 `Stop()` 来进行优雅退出：

```go
type IApp interface {
    //...
    Stop()
}

type App struct {
    running   bool
    stop      chan struct{}
    onStopped func()
}

func New() *App {
    app := &App{
        running: true,
        stop:    make(chan struct{}),
    }
    go watch()
    return app
}

func (app *App) watch() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-app.stop:
            if app.onStopped != nil {
                app.onStopped()
            }
            return
        case <-ticker.C:
            // do something
        }
    }
}

func (app *App) Stop() {
    if !app.running {
        return
    }
    close(app.stop)
}
```

这种方式除了需要在程序终止之前显式调用 `Stop()`，没有啥问题，事实上在业务层也推荐这种显式的处理方式。

## 问题场景：cache 

比如现在想实现一个 cache 模块，接口很简单：

```go
type Cache interface {
    Get(key string) (interface{}, bool)
    Set(key string, value interface{})
}
```

由于需要定时清理过期的缓存，会使用一个后台 goroutine 执行清理工作。但这对使用者来说应该是透明的，然而有时会出现意料之外的情况：

```go
func main() {
    c := cache.New()
    c.Set("key1", obj)
    val, exist := c.Get("key1")
    // ...
    c = nil
    // do other things
}
```

在使用者看来，cache 已经没有引用了，会在 GC 时被回收。但实际上由于后台 goroutine 的存在，cache 始终不能满足不可达的条件，也就不会被 GC 回收，从而产生了内存泄露。

解决方法当然可以显式增加一个 `Close()` 方法，靠 channel 通知关闭 goroutine。但这无疑增加了使用成本，而且也不能避免使用者忘记调用 `Close()` 的场景。


有没有更好的方式，不需要用户显式关闭，在检测到没有引用之后主动终止 goroutine？当然有。`runtime.SetFinalizer` 可以帮助我们达到这个目的。


## SetFinalizer 原理

```go
func SetFinalizer(obj interface{}, finalizer interface{})
```

> SetFinalizer sets the finalizer associated with obj to the provided finalizer function.
> When the garbage collector finds an unreachable block with an associated finalizer,
> it clears the association and runs finalizer(obj) in a separate goroutine.
> This makes obj reachable again, but now without an associated finalizer. Assuming that SetFinalizer is not called again,
> the next time the garbage collector sees that obj is unreachable, it will free obj.

上面是官方文档对 SetFinalizer 的一些解释，主要含义是对象可以关联一个 SetFinalizer 函数， 当 GC 检测到 unreachable 对象有关联的 SetFinalizer 函数时，会执行关联的 SetFinalizer 函数， 同时取消关联。 这样当下一次 GC 的时候，对象重新处于 unreachable 状态并且没有 SetFinalizer 关联， 就会被回收。


仔细看文档，还有几个需要注意的点：

+ 即使程序正常结束或者发生错误，在对象被 GC 选中并回收之前，SetFinalizer 都不会执行。所以不要在 SetFinalizer 中执行将内存内容 flush 到磁盘这类操作。
+ SetFinalizer 最大的问题是延长了对象生命周期。在第一次回收时执行 Finalizer 函数，目标对象重新变成可达状态，直到第二次才真正「销毁」。这对于有大量对象分配的高并发场景，可能会造成很大麻烦。
+ 指针构成的「循环引用」加上 `runtime.SetFinalizer` 会导致内存泄露。

## 正确姿势

如何利用 SetFinalizer 来清理 cache 后台 goroutine？istio 的 lrucache 给了我们一种巧妙的思路：

```go
type lruWrapper struct {
    *lruCache
}

// We return a 'see-through' wrapper for the real object such that
// the finalizer can trigger on the wrapper. We can't set a finalizer
// on the main cache object because it would never fire, since the
// evicter goroutine is keeping it alive
result := &lruWrapper{c}
runtime.SetFinalizer(result, func(w *lruWrapper) {
    w.stopEvicter <- true
    w.evicterTerminated.Wait()
})
```

在 lrucache 外面加上一层 wrapper，lrucache 作为 wrapper 的匿名字段存在，并在 wrapper 上注册 SetFinalizer 函数来终止后台 goroutine。由于后台 goroutine 和 lrucache 关联，当没有引用指向 wrapper 时，GC 就会执行关联的 SetFinalizer 终止 lrucache 的后台 goroutine，最终 lrucache 也会变成不可达状态被 GC 回收。

### 完整实现

```go
type Cache = *wrapper

type wrapper struct {
    *cache
}

type cache struct {
    content   string
    stop      chan struct{}
    onStopped func()
}

func newCache() *cache {
    return &cache{
        content: "some thing",
        stop:    make(chan struct{}),
    }
}

func NewCache() Cache {
    w := &wrapper{
        cache: newCache(),
    }
    go w.cache.run()
    runtime.SetFinalizer(w, (*wrapper).stop)
    return w
}

func (w *wrapper) stop() {
    w.cache.stop()
}

func (c *cache) run() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // do some thing
        case <-c.stop:
            if c.onStopped != nil {
                c.onStopped()
            }
            return
        }
    }
}

func (c *cache) stop() {
    close(c.stop)
}
```

对于对象是否被回收， 最靠谱的方式就是靠test来检测并保证这一行为：

```go
func TestFinalizer(t *testing.T) {
    s := assert.New(t)

    w := NewCache()
    var cnt int = 0
    stopped := make(chan struct{})
    w.onStopped = func() {
        cnt++
        close(stopped)
    }

    s.Equal(0, cnt)

    w = nil

    runtime.GC()

    select {
    case <-stopped:
    case <-time.After(10 * time.Second):
        t.Fail()
    }

    s.Equal(1, cnt)
}
```

事实上，在基础库中 SetFinalizer 主要的使用场景是减少用户错误使用导致的资源泄露。比如 `os.NewFile()` 和 `net.FD()` 都注册了 finalizer 来避免用户由于忘记调用 `Close` 导致的 fd leak，有兴趣的读者可以去看一下相关的代码。