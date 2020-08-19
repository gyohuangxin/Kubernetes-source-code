## WorkQueue
Kubernetes中的工作队列（workqueue）和普通FIFO（先进先出）队列相比，实现略微复杂，它的主要功能在于标记与去重，并实现如下特性。

· 有序

· 去重

· 并发性

· 标记机制

· 通知机制

· 延迟

· 限速

· Metric

WorkQueue支持三种队列，并提供了三种接口：

· Interface： FIFO队列接口，支持去重机制。

· DelayingInterface： 延迟队列接口，基于Interface接口封装，延迟一段时间后将元素存入队列。

· RateLimitingInterface： 限速队列接口，基于DelayingInterface进行封装，支持元素存入队列时进行速率限制。

## FIFO队列

FIFO提供如下方法：

```
type Interface interface {
    Add(item interface{}) // 给队列添加元素（item）,可以是任意类型元素。
    Len() int // 返回当前队列的长度。
    Get() (item interface{}, shutdown bool) // 获取队列头部的一个元素。
    Done(item interface{}) // 标记队列中该元素已被处理。 
    ShutDown() // 关闭队列。
    ShuttingDown() bool // 查询队列是否关闭。    
}
```

FIFO队列数据结构如下：

```
type Type struce {
    queue []t
    dirty set
    processing set
    cond *sync.Cond
    shuttingDown bool
    metrics queueMetrics
    unfinishedWorkUpdatePeriod time.Duration
    clock                   clock.Clock
}
```

FIFO队列中最主要的字段有queue、dirty和proccessing。其中queue字段是实际存储元素的地方，它是slice结构的，用于保证元素有序；dirty字段除了能保证去重，还能保证并发情况下元素只被处理一次；process字段用于标记机制，标记一个元素是否正在被处理。dirty和proccessing字段都是用Hash Map数据结构实现的，所以不需要考虑无序，只需要保证去重即可。

## 延迟队列
基于FIFO队列接口封装，在原有功能上增加AddAfter方法，其原理是延迟一段时间后再将元素插入FIFO队列，延迟队列数据结构如下：

```
type DelayingInterface interface {
    Interface
    AddAfter(item interface{}, duration time.Duration)
}

type delayingType struct {
    Interface
    clock clock.Clock
    stopCh chan struct{}
    heartbeat clock.Ticker
    waitingForAddCh chan *waitFor
    metrics retryMetrics
    deprecateMetrics retryMetrics
}
```

delayingType结构中最重要的字段是waitingForAddCH, 其默认初始大小为1000，通过AddAfter方法插入元素事，是非阻塞状态的，只有当插入元素大于或等于1000时，延迟队列才会处于阻塞状态。waitingForAddCh字段中的数据通过goroutine运行的waitingLoop函数持久运行。