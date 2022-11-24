# TimeWheel

## 时间轮原理分析

技术有时就源于生活，例如排队买票可以想到队列，公司的组织关系可以理解为树等，而时间轮算法的设计思想就来源于钟表。如下图所示，时间轮可以理解为一种环形结构，像钟表一样被分为多个 slot 槽位。每个 slot 代表一个时间段，每个 slot 中可以存放多个任务，使用的是链表结构保存该时间段到期的所有任务。时间轮通过一个时针随着时间一个个 slot 转动，并执行 slot 中的所有到期任务。

![image-20221124152440380](https://image.fanyao666.online/wechat/image-20221124152440380.png)

任务是如何添加到时间轮当中的呢？可以根据任务的到期时间进行取模，然后将任务分布到不同的 slot 中。如上图所示，时间轮被划分为 8 个 slot，每个 slot 代表 1s，当前时针指向 2。假如现在需要调度一个 3s 后执行的任务，应该加入 2+3=5 的 slot 中；如果需要调度一个 12s 以后的任务，需要等待时针完整走完一圈 round 零 4 个 slot，需要放入第 (2+12)%8=6 个 slot。

那么当时针走到第 6 个 slot 时，怎么区分每个任务是否需要立即执行，还是需要等待下一圈 round，甚至更久时间之后执行呢？所以我们需要把 round 信息保存在任务中。例如图中第 6 个 slot 的链表中包含 3 个任务，第一个任务 round=0，需要立即执行；第二个任务 round=1，需要等待 1*8=8s 后执行；第三个任务 round=2，需要等待 2*8=8s 后执行。所以当时针转动到对应 slot 时，只执行 round=0 的任务，slot 中其余任务的 round 应当减 1，等待下一个 round 之后执行。

上面介绍了时间轮算法的基本理论，可以看出时间轮有点类似 HashMap，如果多个任务如果对应同一个 slot，处理冲突的方法采用的是拉链法。在任务数量比较多的场景下，适当增加时间轮的 slot 数量，可以减少时针转动时遍历的任务个数。

时间轮定时器最大的优势就是，任务的新增和取消都是 O(1) 时间复杂度，而且只需要一个线程就可以驱动时间轮进行工作。HashedWheelTimer 是 Netty 中时间轮算法的实现类，下面我就结合 HashedWheelTimer 的源码详细分析时间轮算法的实现原理。

## 使用场景

- 场景：

  - 时间驱动处理场景：如整点发送优惠券，每天定时更新收益，每天定时刷新标签数据和人群数据
  - 批量处理数据：如按月批量统计报表数据，批量更新某些数据状态，实时性要求不高

  - 异步执行解耦：如先反馈用户操作状态，后台异步执行较耗时的数据操作，以实现异步逻辑解耦

- 时间维度：

  - 定时任务：每天定时执行某任务
  - 延时任务：延迟多少分钟执行某任务

## Golang实现(单层时间轮)

### 初始化

定义时间轮结构

```go
type TimeWheel struct {
	interval time.Duration // 时间轮定时转动时间
	ticker   *time.Ticker  // 时间轮定时转动任务
	slots    []*list.List  // 链表数组

	timer             map[string]*location // 存储任务的位置信息
	currentPos        int                  // 当前指针位置
	slotNum           int                  // 插槽个数
	addTaskChannel    chan task
	removeTaskChannel chan string
	stopChannel       chan bool
}

type task struct {
	delay  time.Duration // 多久执行
	circle int           // 圈数
	key    string        // 任务key
	job    func()        // 执行任务
}

// 任务的定位
type location struct {
	slot  int           // 插槽位置
	etask *list.Element // 任务的链表元素
}

```

结构初始化

```go
func New(interval time.Duration, slotNum int) *TimeWheel {
	if interval <= 0 || slotNum <= 0 {
		return nil
	}
	tw := &TimeWheel{
		interval:          interval,
		slots:             make([]*list.List, slotNum),
		timer:             make(map[string]*location),
		currentPos:        0,
		slotNum:           slotNum,
		addTaskChannel:    make(chan task),
		removeTaskChannel: make(chan string),
		stopChannel:       make(chan bool),
	}
	tw.initSlots()
	return tw
}
// 有多少插槽new多少个链表
func (tw *TimeWheel) initSlots() {
	for i := 0; i < tw.slotNum; i++ {
		tw.slots[i] = list.New()
	}
}
```

启动定时任务转动时针

```go
func (tw *TimeWheel) Start() {
	tw.ticker = time.NewTicker(tw.interval)
	go tw.start()
}
func (tw *TimeWheel) start() {
	for {
		select {
		case <-tw.ticker.C:
			tw.tickHandler() // ticker周期性发送时间，转动时针
		case task := <-tw.addTaskChannel:
			tw.addTask(&task)// 处理添加任务
		case key := <-tw.removeTaskChannel:
			tw.removeTask(key)// 处理删除任务
		case <-tw.stopChannel:
			tw.ticker.Stop()// 停止时间轮
			return
		}
	}
}
```

### 添加任务

```go
func (tw *TimeWheel) AddJob(delay time.Duration, key string, job func()) {
	if delay < 0 {
		return
	}
	tw.addTaskChannel <- task{delay: delay, key: key, job: job}
}

func (tw *TimeWheel) addTask(task *task) {
	// 根据key确定插槽位置
	pos, circle := tw.getPositionAndCircle(task.delay)
	task.circle = circle

    // 指定插槽位置的链表最后添加任务元素
	e := tw.slots[pos].PushBack(task)
    // 任务的定位信息
	loc := &location{
		slot:  pos,
		etask: e,
	}
	if task.key != "" {
		// 之前有同名任务存在则删除
		_, ok := tw.timer[task.key]
		if ok {
			tw.removeTask(task.key)
		}
	}
	tw.timer[task.key] = loc
}

func (tw *TimeWheel) getPositionAndCircle(delay time.Duration) (pos int, circle int) {
	delaySeconds := int(delay.Seconds())
	intervalSeconds := int(tw.interval.Seconds())
	circle = int(delaySeconds / intervalSeconds / tw.slotNum) // 计算圈数
	pos = int(tw.currentPos+delaySeconds/intervalSeconds) % tw.slotNum // 计算插槽位置
	return
}
```

### 删除任务

```go
func (tw *TimeWheel) RemoveJob(key string) {
	tw.removeTaskChannel <- key
}

func (tw *TimeWheel) removeTask(key string) {
    // 通过任务名找到任务定位信息
	loc, ok := tw.timer[key]
	if !ok {
		return
	}
    // 删除插槽链表的任务元素
	l := tw.slots[loc.slot]
	l.Remove(loc.etask)
	delete(tw.timer, key)
}
```

### 停止时间轮

```go
func (tw *TimeWheel) Stop() {
	tw.stopChannel <- true
}
```

### 封装

```go

var tw = New(time.Second, 3600)

func init() {
	tw.Start()
}

func At(at time.Time, key string, job func()) {
	tw.AddJob(at.Sub(time.Now()), key, job)
}

func Delay(delay time.Duration, key string, job func()) {
	tw.AddJob(delay, key, job)
}

func Cancel(key string) {
	tw.RemoveJob(key)
}

func Stop() {
	tw.Stop()
}
```

### 测试

```go
func TestAt(t *testing.T) {
	At(time.Now().Add(5*time.Second), "at", func() {
		println(time.Now().String() + " at")
	})
	time.Sleep(7 * time.Second)
}

func TestDelay(t *testing.T) {
	ch := make(chan time.Time)
	now := time.Now()
	Delay(5*time.Second, "delay", func() {
		println("delay")
		ch <- time.Now()
	})
	execAt := <-ch
	if execAt.Sub(now) < 5*time.Second {
		t.Error()
	}
}

func TestCancel(t *testing.T) {
	now := time.Now()
	At(now.Add(5*time.Second), "cancel", func() {
		println("cancel")
	})
	Cancel("cancel")
	time.Sleep(7 * time.Second)
}
```

