作者：<https://github.com/daemon365/p/18593573>



---

* [传统 OS 线程](#_caption_0)
* [GMP 模型](#_caption_1)
* [数据结构](#_caption_2)
	+ [G](#_caption0)
	+ [M](#_caption1)
	+ [P](#_caption2)
	+ [全局队列](#_caption3)
* [调度](#_caption_3)
	+ [主动调度](#_caption4)
	+ [阻塞调度](#_caption5)
	+ [抢占调度](#_caption6)
	+ [调度代码](#_caption7)
	+ [syscall](#_caption8)
* [goroutine 切换通用寄存器问题](#_caption_4):[wgetcloud加速器官网下载](https://longdu.org)


**本身涉及到的 go 代码 都是基于 go 1\.23\.0 版本**


## 传统 OS 线程


线程是 CPU 的最小调度单位，CPU 通过不断切换线程来实现多任务的并发。这会引发一些问题（对于用户角度）：


1. 线程的创建和销毁等是昂贵的，因为要不断在用户空间和内核空间切换。
2. 线程的调度是由操作系统负责的，用户无法控制。而操作系统又可能不知道线程已经 IO 阻塞，导致线程被调度，浪费 CPU 资源。
3. 线程的栈是很大的，最新版 linux 默认是 8M，会引起内存浪费。
4. ......


所以，最简单的办法就是复用线程，go 中使用的是 M:N 模型，即 M 个 OS 线程对应 N 个 任务。


## GMP 模型


1. G


goroutine, 一个 goroutine 代表一个任务。它有自己的栈空间，默认是 2K，栈空间可以动态增长。方式就是把旧的栈空间复制到新的栈空间，然后释放旧的栈空间。它的栈是在 heap （对于 OS） 上分配的。


2. M


machine, 一个 M 代表一个 OS 线程。


3. P


processor, 一个 P 代表一个逻辑处理器，它维护了一个 goroutine 队列。P 会把 goroutine 分配给 M，M 会执行 goroutine。默认的大小为 CPU 核心数。


![](https://img2024.cnblogs.com/blog/2344773/202412/2344773-20241208173113583-400299015.png)


## 数据结构


### G


结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：



```


|  | type g struct { |
| --- | --- |
|  | // goroutine 的栈 两个地址，分别是栈的起始地址和结束地址 |
|  | stack       stack |
|  | // 绑定的m |
|  | m         *m |
|  | // goroutine 被调度走保存的中间状态 |
|  | sched     gobuf |
|  | // goroutine 的状态 |
|  | atomicstatus atomic.Uint32 |
|  | } |
|  |  |
|  | type gobuf struct { |
|  | sp   uintptr // stack pointer 栈指针 |
|  | pc   uintptr // program counter 程序要从哪里开始执行 |
|  | g    guintptr // goroutine 的 指针 |
|  | ctxt unsafe.Pointer // 保存的上下文 |
|  | ret  uintptr // 返回地址 |
|  | lr   uintptr // link register |
|  | bp   uintptr // base pointer 栈的基地址 |
|  | } |


```

#### goroutine 状态



```


|  | // defined constants |
| --- | --- |
|  | const ( |
|  | // 未初始化 |
|  | _Gidle = iota // 0 |
|  |  |
|  | // 准备好了 可以被 P 调度 |
|  | _Grunnable // 1 |
|  |  |
|  | // 正在执行中 |
|  | _Grunning // 2 |
|  |  |
|  | // 正在执行系统调用 |
|  | _Gsyscall // 3 |
|  |  |
|  | // 正在等待 例如 channel network 等 |
|  | _Gwaiting // 4 |
|  |  |
|  | // 没有被使用 为了兼容性 |
|  | _Gmoribund_unused // 5 |
|  |  |
|  | // 未使用的 goroutine |
|  | // 1. 可能初始化了但是没有被使用 |
|  | // 2. 因为会复用未扩栈的 goroutine 所以也可能上次使用完了 还没继续使用 |
|  | _Gdead // 6 |
|  |  |
|  | // 没有被使用 为了兼容性 |
|  | _Genqueue_unused // 7 |
|  |  |
|  | // 栈扩容中 |
|  | _Gcopystack // 8 |
|  |  |
|  | // 被抢占了 等待到 _Gwaiting |
|  | _Gpreempted // 9 |
|  |  |
|  | // 用于 GC 扫描 |
|  | _Gscan          = 0x1000 |
|  | _Gscanrunnable  = _Gscan + _Grunnable  // 0x1001 |
|  | _Gscanrunning   = _Gscan + _Grunning   // 0x1002 |
|  | _Gscansyscall   = _Gscan + _Gsyscall   // 0x1003 |
|  | _Gscanwaiting   = _Gscan + _Gwaiting   // 0x1004 |
|  | _Gscanpreempted = _Gscan + _Gpreempted // 0x1009 |
|  | ) |


```


```


|  | func malg(stacksize int32) *g { |
| --- | --- |
|  | newg := new(g) |
|  | // 分配 runtime 栈 |
|  | if stacksize >= 0 { |
|  | stacksize = round2(stackSystem + stacksize) |
|  | systemstack(func() { |
|  | newg.stack = stackalloc(uint32(stacksize)) |
|  | }) |
|  | newg.stackguard0 = newg.stack.lo + stackGuard |
|  | newg.stackguard1 = ^uintptr(0) |
|  | *(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0 |
|  | } |
|  | return newg |
|  | } |


```

状态流转:


![](https://img2024.cnblogs.com/blog/2344773/202412/2344773-20241208171758824-1397522258.png)


1. 如果 groutine 还未初始化，那么状态是 `_Gidle`
2. 初始化完毕是 `_Gdead`
3. 当被调用 go func() 时，状态变为 `_Grunnable`
4. 当被调度到 M 上执行时，状态变为 `_Grunning`
5. 执行完毕后，状态变为 `_Gdead`
6. 如果 goroutine 阻塞，状态变为 `_Gwaiting` 等待阻塞完毕 状态再变为 `_Grunnable` 等待调度
7. 如果 goroutine 被抢占 （gc 要 STW 时），状态变为 `_Gpreempted` 等待变成 `_Gwaiting`
8. 如果发生系统调用，状态变为 `_Gsyscall` 如果很快完成（10ms） 状态会变为 `_Grunning` 继续执行 否则会变为 `_Grunnable` 等待调度
9. 如果发生栈扩容，状态变为 `_Gcopystack` 等待栈扩容完毕 状态变为 `_Grunnable` 等待调度


### M


结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：



```


|  | type m struct { |
| --- | --- |
|  | g0      *g |
|  | // 寄存器上下文 |
|  | morebuf gobuf |
|  | // tls 是线程本地存储 用于存储 M 相关的线程本地数据 包括当前 G 的引用等重要信息 |
|  | tls           [tlsSlots]uintptr |
|  | // 现在正在执行的 goroutine |
|  | curg          *g |
|  |  |
|  | // 1. 正常执行： p 有效 |
|  | // 2. 系统调用前： p -> oldp |
|  | // 3. 系统调用中： p == nil |
|  | // 4. 系统调用返回： 尝试重新获取 oldp |
|  | p             puintptr |
|  | nextp         puintptr |
|  | oldp          puintptr |
|  | } |


```

#### M 的创建



```


|  | func newm(fn func(), pp *p, id int64) { |
| --- | --- |
|  | // 禁止被抢占 |
|  | acquirem() |
|  | // 分配 M 结构体 并添加列表中 |
|  | mp := allocm(pp, fn, id) |
|  | // 设置 nextP m 会尽量与之绑定 |
|  | mp.nextp.set(pp) |
|  | mp.sigmask = initSigmask |
|  | // ... |
|  |  |
|  | // 创建 M |
|  | newm1(mp) |
|  | // 释放 m 的锁定状态 |
|  | releasem(getg().m) |
|  | } |
|  |  |
|  | func allocm(pp *p, fn func(), id int64) *m { |
|  | // ... 加锁解锁 |
|  | // 如果当前 M 没有绑定 P，临时借用传入的 P |
|  | if gp.m.p == 0 { |
|  | acquirep(pp) // temporarily borrow p for mallocs in this function |
|  | } |
|  |  |
|  | // 处理空闲的 M |
|  | if sched.freem != nil { |
|  |  |
|  | } |
|  |  |
|  | // 创建 M |
|  | mp := new(m) |
|  | mp.mstartfn = fn |
|  | mcommoninit(mp, id) |
|  |  |
|  | // 对于 cgo 或者特定的操作系统 使用系统分配的栈 否则使用 go runtime 的栈 |
|  | if iscgo || mStackIsSystemAllocated() { |
|  | mp.g0 = malg(-1) |
|  | } else { |
|  | mp.g0 = malg(16384 * sys.StackGuardMultiplier) |
|  | } |
|  | mp.g0.m = mp |
|  | // 清理临时借用的 P |
|  | if pp == gp.m.p.ptr() { |
|  | releasep() |
|  | } |
|  | return mp |
|  | } |
|  |  |


```


```


|  | // newm1 -> newosproc |
| --- | --- |
|  | func newosproc(mp *m) { |
|  | // 栈顶指针 |
|  | stk := unsafe.Pointer(mp.g0.stack.hi) |
|  |  |
|  | // 信号屏蔽 |
|  | var oset sigset |
|  | sigprocmask(_SIG_SETMASK, &sigset_all, &oset) |
|  | // 重试的系统调用 |
|  | ret := retryOnEAGAIN(func() int32 { |
|  | // 创建新线程 是汇编代码 可以找去看看 |
|  | r := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(abi.FuncPCABI0(mstart))) |
|  | if r >= 0 { |
|  | return 0 |
|  | } |
|  | return -r |
|  | }) |
|  | // 恢复信号 |
|  | sigprocmask(_SIG_SETMASK, &oset, nil) |
|  | // ... |
|  | } |


```


```


|  | // mstart 启动 M 是一个汇编代码 |
| --- | --- |
|  | TEXT runtime·mstart(SB),NOSPLIT|TOPFRAME|NOFRAME,$0 |
|  | CALL	runtime·mstart0(SB) // 调用 mstart0 |
|  | RET // not reached |
|  |  |
|  | // mstart0 -> mstart1 |
|  | func mstart1() { |
|  | gp := getg() |
|  |  |
|  | if gp != gp.m.g0 { |
|  | throw("bad runtime·mstart") |
|  | } |
|  |  |
|  | // 保存调度信息 |
|  | gp.sched.g = guintptr(unsafe.Pointer(gp)) |
|  | gp.sched.pc = getcallerpc() |
|  | gp.sched.sp = getcallersp() |
|  |  |
|  | // 初始化 |
|  | asminit() |
|  | minit() |
|  | // 主线程初始一些东西 |
|  | if gp.m == &m0 { |
|  | mstartm0() |
|  | } |
|  |  |
|  |  |
|  | // 调度 |
|  | schedule() |
|  | } |
|  |  |
|  |  |
|  | // 初始化信号 之后抢占那块会介绍 |
|  | // 只有主线程需要初始化的原因是 其他线程是 clone 而来 而且包括了 _CLONE_SIGHAND 会继承这些 |
|  | func mstartm0() { |
|  | // ... |
|  | initsig(false) |
|  | } |


```

g0： 一个特殊的 g 用于执行调度任务 它未使用 go runtime 的 stack 而是使用 os stack
流程大概为用户态的 g \-\> g0 调度 \-\> 用户的其他 g


### P


结构体在 `src/runtime/runtime2.go` 中定义，主要介绍一些重要的字段：



```


|  | type p struct { |
| --- | --- |
|  | // p 的状态 |
|  | status      uint32 |
|  | // 分配内存使用 每个p 都有的目的是少加锁 |
|  | mcache      *mcache |
|  | // 定长的 queue 用于存储 goroutine |
|  | runqhead uint32 |
|  | runqtail uint32 |
|  | runq     [256]guintptr |
|  | //  下个运行的 goroutine 主要用来快速调度 比如从 chan 读取数据，把 g 放到 runnext 中 当完成读取时 直接从 runnext 中取出来执行 |
|  | runnext guintptr |
|  |  |
|  | } |


```

状态：



```


|  | const ( |
| --- | --- |
|  | // 空闲 |
|  | _Pidle = iota |
|  |  |
|  | // 正在运行中 |
|  | _Prunning |
|  |  |
|  | // 正在执行系统调用 |
|  | _Psyscall |
|  |  |
|  | // GC 停止 |
|  | _Pgcstop |
|  |  |
|  | // 死亡状态 |
|  | _Pdead |
|  | ) |


```

#### P的创建



```


|  | // 程序启动 |
| --- | --- |
|  | TEXT main(SB),NOSPLIT,$-8 |
|  | JMP	runtime·rt0_go(SB) |
|  |  |
|  | TEXT runtime·rt0_go(SB),NOSPLIT|NOFRAME|TOPFRAME,$0 |
|  | // ... |
|  | CALL	runtime·schedinit(SB) |
|  |  |
|  | func schedinit() { |
|  | // ... |
|  | if procresize(procs) != nil { |
|  | throw("unknown runnable goroutine during bootstrap") |
|  | } |
|  | } |
|  |  |
|  | // nprocs 是 process 数 默认是 cpu 个数 |
|  | func procresize(nprocs int32) *p { |
|  | // ... |
|  |  |
|  | // 扩容 allp 加入未初始化的 P |
|  | if nprocs > int32(len(allp)) { |
|  | // 。。。 |
|  | } |
|  |  |
|  | // 初始化所有新建的 P |
|  | for i := old; i < nprocs; i++ { |
|  | pp := allp[i] |
|  | if pp == nil { |
|  | pp = new(p) |
|  | } |
|  | pp.init(i) |
|  | atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp)) |
|  | } |
|  |  |
|  | // 处理 p 的状态 |
|  | gp := getg() |
|  | if gp.m.p != 0 && gp.m.p.ptr().id < nprocs { |
|  | // continue to use the current P |
|  | gp.m.p.ptr().status = _Prunning |
|  | gp.m.p.ptr().mcache.prepareForSweep() |
|  | } else { |
|  | // ... |
|  | } |
|  |  |
|  | // g.m.p is now set, so we no longer need mcache0 for bootstrapping. |
|  | mcache0 = nil |
|  |  |
|  | // 清理多余的 P |
|  | for i := nprocs; i < old; i++ { |
|  | pp := allp[i] |
|  | pp.destroy() |
|  | // can't free P itself because it can be referenced by an M in syscall |
|  | } |
|  |  |
|  | // 裁剪 allp 切片 |
|  | if int32(len(allp)) != nprocs { |
|  | lock(&allpLock) |
|  | allp = allp[:nprocs] |
|  | idlepMask = idlepMask[:maskWords] |
|  | timerpMask = timerpMask[:maskWords] |
|  | unlock(&allpLock) |
|  | } |
|  |  |
|  | // 重新分配 P |
|  | var runnablePs *p |
|  | for i := nprocs - 1; i >= 0; i-- { |
|  | pp := allp[i] |
|  | if gp.m.p.ptr() == pp { |
|  | continue |
|  | } |
|  | pp.status = _Pidle |
|  | if runqempty(pp) { |
|  | pidleput(pp, now) |
|  | } else { |
|  | pp.m.set(mget()) |
|  | pp.link.set(runnablePs) |
|  | runnablePs = pp |
|  | } |
|  | } |
|  | stealOrder.reset(uint32(nprocs)) |
|  | var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32 |
|  | atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs)) |
|  | if old != nprocs { |
|  | // Notify the limiter that the amount of procs has changed. |
|  | gcCPULimiter.resetCapacity(now, nprocs) |
|  | } |
|  | return runnablePs |
|  | } |
|  |  |
|  | func (pp *p) init(id int32) { |
|  | // ... |
|  | // 分配 cache |
|  | if pp.mcache == nil { |
|  | if id == 0 { |
|  | if mcache0 == nil { |
|  | throw("missing mcache?") |
|  | } |
|  | // Use the bootstrap mcache0. Only one P will get |
|  | // mcache0: the one with ID 0. |
|  | pp.mcache = mcache0 |
|  | } else { |
|  | pp.mcache = allocmcache() |
|  | } |
|  | } |
|  | // ... |
|  | } |
|  |  |
|  | func (pp *p) destroy() { |
|  | // 枷锁 确保 stw |
|  | assertLockHeld(&sched.lock) |
|  | assertWorldStopped() |
|  |  |
|  | // 将本地队列中的 goroutine 移到全局队列 |
|  | for pp.runqhead != pp.runqtail { |
|  | pp.runqtail-- |
|  | gp := pp.runq[pp.runqtail%uint32(len(pp.runq))].ptr() |
|  | globrunqputhead(gp) |
|  | } |
|  | if pp.runnext != 0 { |
|  | globrunqputhead(pp.runnext.ptr()) |
|  | pp.runnext = 0 |
|  | } |
|  |  |
|  | // ... |
|  |  |
|  | // 清理 span |
|  | systemstack(func() { |
|  | for i := 0; i < pp.mspancache.len; i++ { |
|  | // Safe to call since the world is stopped. |
|  | mheap_.spanalloc.free(unsafe.Pointer(pp.mspancache.buf[i])) |
|  | } |
|  | pp.mspancache.len = 0 |
|  | lock(&mheap_.lock) |
|  | pp.pcache.flush(&mheap_.pages) |
|  | unlock(&mheap_.lock) |
|  | }) |
|  |  |
|  | // 释放 mcache |
|  | freemcache(pp.mcache) |
|  | pp.mcache = nil |
|  |  |
|  | // ... |
|  | } |
|  |  |


```

### 全局队列



```


|  | type schedt struct { |
| --- | --- |
|  | // 锁 |
|  | lock mutex |
|  |  |
|  | // m 相关配置 |
|  | midle        muintptr |
|  | // ... |
|  |  |
|  | // p 相关配置 |
|  | pidle        puintptr // idle p's |
|  | // ... |
|  |  |
|  | // g 队列 |
|  | runq     gQueue |
|  | runqsize int32 |
|  |  |
|  | // ... |
|  |  |
|  | // G 对象池 |
|  | gFree struct { |
|  | lock    mutex |
|  | stack   gList // Gs with stacks |
|  | noStack gList // Gs without stacks |
|  | n       int32 |
|  | } |
|  |  |
|  | // ... |
|  | } |
|  |  |


```

P 的空闲列表： M 获取 P 的时候拿到
M 的空闲列表： 线程的创建于销毁代价是很大的 为了复用性


## 调度


go 有三种进行到调度的方式：


1. 用户 goroutine 主动执行 runtime.Gosched() 会把当前 goroutine 放到队列中等待调度
2. 用户 goroutine 阻塞，例如 channel 读写，网络 IO 等 会主动调用修改自己状态并切换到 g0 执行调度任务
3. go runtime 中有个 OS 线程 （名称是 sysmon） 检测到 goroutine 超时（上次执行到现在超过 10ms）那就会给线程发信号 使其切换到 g0 执行调度任务


***为什么 sysmon 使用物理线程而不是 goroutine 呢？***


因为所有 p 上正在执行的 g 都阻塞住了 比如 `for {}` 那么其他的 g 永远无法执行了包括负责检测的 sysmon


### 主动调度



```


|  | func Gosched() { |
| --- | --- |
|  | checkTimeouts() |
|  | mcall(gosched_m) |
|  | } |


```

### 阻塞调度



```


|  | func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceReason traceBlockReason, traceskip int) { |
| --- | --- |
|  | if reason != waitReasonSleep { |
|  | checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy |
|  | } |
|  | mp := acquirem() |
|  | gp := mp.curg |
|  | status := readgstatus(gp) |
|  | if status != _Grunning && status != _Gscanrunning { |
|  | throw("gopark: bad g status") |
|  | } |
|  | mp.waitlock = lock |
|  | mp.waitunlockf = unlockf |
|  | gp.waitreason = reason |
|  | mp.waitTraceBlockReason = traceReason |
|  | mp.waitTraceSkip = traceskip |
|  | releasem(mp) |
|  | // can't do anything that might move the G between Ms here. |
|  | mcall(park_m) |
|  | } |


```

### 抢占调度



```


|  | // m 在 start 的时候会注册一些信号处理函数 |
| --- | --- |
|  | func initsig(preinit bool) { |
|  | for i := uint32(0); i < _NSIG; i++ { |
|  | // ... |
|  | setsig(i, abi.FuncPCABIInternal(sighandler)) |
|  | } |
|  | } |
|  |  |
|  | // sighandler -> doSigPreempt -> asyncPreempt （去汇编代码里找） -> asyncPreempt2 |
|  | func asyncPreempt2() { |
|  | gp := getg() |
|  | gp.asyncSafePoint = true |
|  | if gp.preemptStop { |
|  | mcall(preemptPark) |
|  | } else { |
|  | mcall(gopreempt_m) |
|  | } |
|  | gp.asyncSafePoint = false |
|  | } |
|  |  |
|  |  |
|  | // sysmon 发信号 |
|  | // sysmon -> retake -> preemptone -> preemptM |
|  | func preemptM(mp *m) { |
|  | if mp.signalPending.CompareAndSwap(0, 1) { |
|  | // ... |
|  | signalM(mp, sigPreempt) |
|  | } |
|  | } |
|  |  |
|  | func signalM(mp *m, sig int) { |
|  | tgkill(getpid(), int(mp.procid), sig) |
|  | } |
|  |  |
|  | // 代码在在汇编里 就是对线程发送信号 系统调用 |
|  | func tgkill(tgid, tid, sig int) |


```

### 调度代码


可以看到调度代码都是通过 mcall 调用的，mcall 会切换到 g0 执行调度任务 如果参数的函数不太一样 但是都是处理一些状态信息等，最好都会执行到 schedule 函数。



```


|  | func schedule() { |
| --- | --- |
|  | // 核心代码就是选一个 g 去执行 |
|  | gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available |
|  |  |
|  | execute(gp, inheritTime) |
|  | } |


```

findRunnable：



```


|  | func findRunnable() (gp *g, inheritTime, tryWakeP bool) { |
| --- | --- |
|  |  |
|  | // Try to schedule a GC worker. |
|  | if gcBlackenEnabled != 0 { |
|  | gp, tnow := gcController.findRunnableGCWorker(pp, now) |
|  | if gp != nil { |
|  | return gp, false, true |
|  | } |
|  | now = tnow |
|  | } |
|  |  |
|  | if pp.schedtick%61 == 0 && sched.runqsize > 0 { |
|  | lock(&sched.lock) |
|  | gp := globrunqget(pp, 1) |
|  | unlock(&sched.lock) |
|  | if gp != nil { |
|  | return gp, false, false |
|  | } |
|  | } |
|  |  |
|  | // local runq |
|  | if gp, inheritTime := runqget(pp); gp != nil { |
|  | return gp, inheritTime, false |
|  | } |
|  |  |
|  | // global runq |
|  | if sched.runqsize != 0 { |
|  | lock(&sched.lock) |
|  | gp := globrunqget(pp, 0) |
|  | unlock(&sched.lock) |
|  | if gp != nil { |
|  | return gp, false, false |
|  | } |
|  | } |
|  |  |
|  | if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 { |
|  | if list, delta := netpoll(0); !list.empty() { // non-blocking |
|  | gp := list.pop() |
|  | injectglist(&list) |
|  | netpollAdjustWaiters(delta) |
|  | trace := traceAcquire() |
|  | casgstatus(gp, _Gwaiting, _Grunnable) |
|  | if trace.ok() { |
|  | trace.GoUnpark(gp, 0) |
|  | traceRelease(trace) |
|  | } |
|  | return gp, false, false |
|  | } |
|  | } |
|  |  |
|  | // Spinning Ms: steal work from other Ps. |
|  | // |
|  | // Limit the number of spinning Ms to half the number of busy Ps. |
|  | // This is necessary to prevent excessive CPU consumption when |
|  | // GOMAXPROCS>>1 but the program parallelism is low. |
|  | if mp.spinning || 2*sched.nmspinning.Load() < gomaxprocs-sched.npidle.Load() { |
|  | if !mp.spinning { |
|  | mp.becomeSpinning() |
|  | } |
|  |  |
|  | gp, inheritTime, tnow, w, newWork := stealWork(now) |
|  | if gp != nil { |
|  | // Successfully stole. |
|  | return gp, inheritTime, false |
|  | } |
|  | if newWork { |
|  | // There may be new timer or GC work; restart to |
|  | // discover. |
|  | goto top |
|  | } |
|  |  |
|  | now = tnow |
|  | if w != 0 && (pollUntil == 0 || w < pollUntil) { |
|  | // Earlier timer to wait for. |
|  | pollUntil = w |
|  | } |
|  | } |
|  |  |
|  | } |


```

简化了一下代码还是很多 价绍一些这个功能吧


1. 优先执行 GC worker
2. 每 61 次 从全局队列中获取一个 g 去执行 作用是 防止所有 p 的本地队列谁都非常多 导致全局队列的 g 饿死
3. 从本地队列中获取一个 g 去执行 有限使用 runnext
4. 从全局队列中获取一个 g 去执行 并 load 一些到本地队列
5. 如果有网络 IO 准备好了 就从网络 IO 中获取一个 g 去执行 （go 中网络 epoll\_wait 正常情况下使用的阻塞模式）
6. 从其他的 p 中偷取 g 去执行 （cas 保证数据安全）


execute：



```


|  | func execute(gp *g, inheritTime bool) { |
| --- | --- |
|  | // 修改状态 |
|  | casgstatus(gp, _Grunnable, _Grunning) |
|  | // 执行 |
|  | gogo(&gp.sched) |
|  | } |


```

我的 arch 是 amd64 所以代码在 `src/runtime/asm_amd64.s` 中



```


|  | TEXT runtime·gogo(SB), NOSPLIT, $0-8 |
| --- | --- |
|  | MOVQ	buf+0(FP), BX	  // 将 gobuf 指针加载到 BX 寄存器 |
|  | MOVQ	gobuf_g(BX), DX  // 将 gobuf 中保存的 g 指针加载到 DX |
|  | MOVQ	0(DX), CX	  // 检查 g 不为 nil |
|  | JMP	gogo<>(SB) |
|  |  |
|  | TEXT gogo<>(SB), NOSPLIT, $0 |
|  | get_tls(CX) |
|  | MOVQ	DX, g(CX) |
|  | MOVQ	DX, R14		// set the g register |
|  | // 恢复寄存器状态 （sp ret bp ctxt） 执行 |
|  | MOVQ	gobuf_sp(BX), SP	// restore SP |
|  | MOVQ	gobuf_ret(BX), AX |
|  | MOVQ	gobuf_ctxt(BX), DX |
|  | MOVQ	gobuf_bp(BX), BP |
|  | // 加载之后 清空 go 的 gobuf 结构体 为了给 gc 节省压力 |
|  | MOVQ	$0, gobuf_sp(BX) |
|  | MOVQ	$0, gobuf_ret(BX) |
|  | MOVQ	$0, gobuf_ctxt(BX) |
|  | MOVQ	$0, gobuf_bp(BX) |
|  | // 跳转到保存的 PC （程序执行到哪了） 去执行 |
|  | MOVQ	gobuf_pc(BX), BX |
|  | JMP	BX |


```

### syscall


我的 arch 是 and64 操作系统是 linux 所以代码在 `src/runtime/asm_linux_amd64.s` 中



```


|  | TEXT ·SyscallNoError(SB),NOSPLIT,$0-48 |
| --- | --- |
|  | CALL	runtime·entersyscall(SB) |
|  | MOVQ	a1+8(FP), DI |
|  | MOVQ	a2+16(FP), SI |
|  | MOVQ	a3+24(FP), DX |
|  | MOVQ	$0, R10 |
|  | MOVQ	$0, R8 |
|  | MOVQ	$0, R9 |
|  | MOVQ	trap+0(FP), AX	// syscall entry |
|  | SYSCALL |
|  | MOVQ	AX, r1+32(FP) |
|  | MOVQ	DX, r2+40(FP) |
|  | CALL	runtime·exitsyscall(SB) |
|  | RET |


```

系统调用前执行这个函数：



```


|  | func entersyscall() { |
| --- | --- |
|  | fp := getcallerfp() |
|  | reentersyscall(getcallerpc(), getcallersp(), fp) |
|  | } |
|  |  |
|  | func reentersyscall(pc, sp, bp uintptr) { |
|  | // 保存寄存器信息 |
|  | save(pc, sp, bp) |
|  | gp.syscallsp = sp |
|  | gp.syscallpc = pc |
|  | gp.syscallbp = bp |
|  | // 修改 g 状态 |
|  | casgstatus(gp, _Grunning, _Gsyscall) |
|  |  |
|  |  |
|  | if sched.sysmonwait.Load() { |
|  | systemstack(entersyscall_sysmon) |
|  | save(pc, sp, bp) |
|  | } |
|  |  |
|  | if gp.m.p.ptr().runSafePointFn != 0 { |
|  | // runSafePointFn may stack split if run on this stack |
|  | systemstack(runSafePointFn) |
|  | save(pc, sp, bp) |
|  | } |
|  |  |
|  | gp.m.syscalltick = gp.m.p.ptr().syscalltick |
|  | pp := gp.m.p.ptr() |
|  | // 解绑 P 和 M 并设置 oldP 为当前 P 等待系统调用之后重新绑定 |
|  | pp.m = 0 |
|  | gp.m.oldp.set(pp) |
|  | gp.m.p = 0 |
|  | // 修改 P 的状态为 syscall |
|  | atomic.Store(&pp.status, _Psyscall) |
|  | if sched.gcwaiting.Load() { |
|  | systemstack(entersyscall_gcwait) |
|  | save(pc, sp, bp) |
|  | } |
|  |  |
|  | gp.m.locks-- |
|  | } |


```

系统调用后执行这个函数：



```


|  | func exitsyscall() { |
| --- | --- |
|  | // 如果之前保存的oldp不为空 那么重新绑定 |
|  | if exitsyscallfast(oldp) { |
|  | // 设置状态为 runnable 并重新执行 |
|  | casgstatus(gp, _Gsyscall, _Grunning) |
|  | if sched.disable.user && !schedEnabled(gp) { |
|  | // Scheduling of this goroutine is disabled. |
|  | Gosched() |
|  | } |
|  |  |
|  | return |
|  | } |
|  | // 切换到 g0 执行 exitsyscall0 |
|  | mcall(exitsyscall0) |
|  | } |
|  |  |


```


```


|  | func exitsyscall0(gp *g) { |
| --- | --- |
|  | // 修改 g 状态到 _Grunnable 让重新可调度 |
|  | casgstatus(gp, _Gsyscall, _Grunnable) |
|  |  |
|  | // 删除 gm 的绑定 |
|  | dropg() |
|  | lock(&sched.lock) |
|  | // 找个空闲的 p （状态为 _Gidle） 与 M 绑定 |
|  | var pp *p |
|  | if schedEnabled(gp) { |
|  | pp, _ = pidleget(0) |
|  | } |
|  | var locked bool |
|  | if pp == nil { |
|  | // 如果绑定失败了 直接把 g 放到全局队列中 |
|  | globrunqput(gp) |
|  | locked = gp.lockedm != 0 |
|  | } else if sched.sysmonwait.Load() { |
|  | // 如果 sysmon 在等待 那么唤醒它 |
|  | sched.sysmonwait.Store(false) |
|  | notewakeup(&sched.sysmonnote) |
|  | } |
|  | unlock(&sched.lock) |
|  | // 如果找到 p 了 那么就去执行 |
|  | if pp != nil { |
|  | acquirep(pp) |
|  | execute(gp, false) // Never returns. |
|  | } |
|  | if locked { |
|  | // Wait until another thread schedules gp and so m again. |
|  | // |
|  | // N.B. lockedm must be this M, as this g was running on this M |
|  | // before entersyscall. |
|  | stoplockedm() |
|  | execute(gp, false) // Never returns. |
|  | } |
|  | // 如果没有 P 给我这个 M 绑定的话 那么把 M 休眠并加入到 schedlink 队列中  做复用 |
|  | stopm() |
|  | // 直到有新的 g 被调度到这个 M 上 |
|  | schedule() // Never returns. |
|  | } |
|  |  |


```

## goroutine 切换通用寄存器问题


我们知道 goroutine 中的 gobuf 中只保存了 sp pc bp 等寄存器信息，但是 goroutine 切换的时候还有其他通用寄存器，如果中间丢失会引起结果不一致。那么 go 中是怎么保存的呢？


goroutine 切换大体有两种情况


1. 在编译阶段知道 goroutine 可能交出控制权 比如 读写 channel 等待网络 系统调用等
2. goroutine 被抢占了 GC 超时等


对于第一种方式，在编译阶段知道后续会使用哪个寄存器和知道在哪里可能会交出控制权，就会在后续保存这些寄存器。



```


|  | func test(a chan int, b, c int) int { |
| --- | --- |
|  | d := <-a |
|  | return d + b + c |
|  | } |
|  |  |
|  | // go tool compile -S main.go 的汇编代码 |
|  | main.test STEXT size=105 args=0x18 locals=0x20 funcid=0x0 align=0x0 |
|  | 0x0000 00000 (./main.go:3)	TEXT	main.test(SB), ABIInternal, $32-24 |
|  | // 栈检查 |
|  | 0x0000 00000 (./main.go:3)	CMPQ	SP, 16(R14) |
|  | 0x0004 00004 (./main.go:3)	PCDATA	$0, $-2 |
|  | 0x0004 00004 (./main.go:3)	JLS	68 |
|  | 0x0006 00006 (./main.go:3)	PCDATA	$0, $-1 |
|  | 0x0006 00006 (./main.go:3)	PUSHQ	BP |
|  | 0x0007 00007 (./main.go:3)	MOVQ	SP, BP |
|  | 0x000a 00010 (./main.go:3)	SUBQ	$24, SP |
|  | // 调试使用的 |
|  | 0x000e 00014 (./main.go:3)	FUNCDATA	$0, gclocals·wgcWObbY2HYnK2SU/U22lA==(SB) |
|  | 0x000e 00014 (./main.go:3)	FUNCDATA	$1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB) |
|  | 0x000e 00014 (./main.go:3)	FUNCDATA	$5, main.test.arginfo1(SB) |
|  | 0x000e 00014 (./main.go:3)	FUNCDATA	$6, main.test.argliveinfo(SB) |
|  | 0x000e 00014 (./main.go:3)	PCDATA	$3, $1 |
|  | // 把 BX CX 保存到栈中 |
|  | 0x000e 00014 (./main.go:5)	MOVQ	BX, main.b+48(SP) |
|  | 0x0013 00019 (./main.go:5)	MOVQ	CX, main.c+56(SP) |
|  | 0x0018 00024 (./main.go:5)	PCDATA	$3, $2 |
|  | // 初始化临时变量 并 chanrecv1 接受 chan 数据 |
|  | 0x0018 00024 (./main.go:4)	MOVQ	$0, main..autotmp_5+16(SP) |
|  | 0x0021 00033 (./main.go:4)	LEAQ	main..autotmp_5+16(SP), BX |
|  | 0x0026 00038 (./main.go:4)	PCDATA	$1, $1 |
|  | 0x0026 00038 (./main.go:4)	CALL	runtime.chanrecv1(SB) |
|  | // 恢复接受 chan 之前入栈的寄存器 |
|  | 0x002b 00043 (./main.go:5)	MOVQ	main.b+48(SP), CX |
|  | 0x0030 00048 (./main.go:5)	ADDQ	main..autotmp_5+16(SP), CX |
|  | 0x0035 00053 (./main.go:5)	MOVQ	main.c+56(SP), DX |
|  | 0x003a 00058 (./main.go:5)	LEAQ	(DX)(CX*1), AX |
|  | 0x003e 00062 (./main.go:5)	ADDQ	$24, SP |
|  | 0x0042 00066 (./main.go:5)	POPQ	BP |
|  | 0x0043 00067 (./main.go:5)	RET |
|  | 0x0044 00068 (./main.go:5)	NOP |
|  | // 处理扩容栈相关的代码 |
|  | 0x0044 00068 (./main.go:3)	PCDATA	$1, $-1 |
|  | 0x0044 00068 (./main.go:3)	PCDATA	$0, $-2 |
|  | 0x0044 00068 (./main.go:3)	MOVQ	AX, 8(SP) |
|  | 0x0049 00073 (./main.go:3)	MOVQ	BX, 16(SP) |
|  | 0x004e 00078 (./main.go:3)	MOVQ	CX, 24(SP) |
|  | 0x0053 00083 (./main.go:3)	CALL	runtime.morestack_noctxt(SB) |
|  | 0x0058 00088 (./main.go:3)	PCDATA	$0, $-1 |
|  | 0x0058 00088 (./main.go:3)	MOVQ	8(SP), AX |
|  | 0x005d 00093 (./main.go:3)	MOVQ	16(SP), BX |
|  | 0x0062 00098 (./main.go:3)	MOVQ	24(SP), CX |
|  | 0x0067 00103 (./main.go:3)	JMP	0 |
|  |  |
|  |  |
|  | func test2(a int, b, c int) int { |
|  | return a + b + c |
|  | } |
|  |  |
|  | main.test2 STEXT nosplit size=9 args=0x18 locals=0x0 funcid=0x0 align=0x0 |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	TEXT	main.test2(SB), NOSPLIT|NOFRAME|ABIInternal, $0-24 |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	FUNCDATA	$0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB) |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	FUNCDATA	$1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB) |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	FUNCDATA	$5, main.test2.arginfo1(SB) |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	FUNCDATA	$6, main.test2.argliveinfo(SB) |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:8)	PCDATA	$3, $1 |
|  | 0x0000 00000 (/home/zhy/code/test1/main.go:9)	LEAQ	(BX)(AX*1), DX |
|  | 0x0004 00004 (/home/zhy/code/test1/main.go:9)	LEAQ	(CX)(DX*1), AX |
|  | 0x0008 00008 (/home/zhy/code/test1/main.go:9)	RET |


```

***从上方可以看到 test2 函数没有保存 BX CX 寄存器，因为编译器知道这个函数不会交出控制权，所以不需要保存这些寄存器。如果调用函数不做参数入栈的话，只用寄存器的话性能会更好。***


那如果是抢占呢，编译阶段肯定是不知道会在哪被抢占的，是怎么恢复要使用的寄存器呢？


处理信号的逻辑：



```


|  | func doSigPreempt(gp *g, ctxt *sigctxt) { |
| --- | --- |
|  | // Check if this G wants to be preempted and is safe to |
|  | // preempt. |
|  | if wantAsyncPreempt(gp) { |
|  | if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok { |
|  | // Adjust the PC and inject a call to asyncPreempt. |
|  | ctxt.pushCall(abi.FuncPCABI0(asyncPreempt), newpc) |
|  | } |
|  | } |
|  |  |
|  | } |


```

代码在 `src/runtime/preempt_amd64.go` 中



```


|  | TEXT ·asyncPreempt(SB),NOSPLIT|NOFRAME,$0-0 |
| --- | --- |
|  | PUSHQ BP |
|  | MOVQ SP, BP |
|  | // Save flags before clobbering them |
|  | PUSHFQ |
|  | // obj doesn't understand ADD/SUB on SP, but does understand ADJSP |
|  | ADJSP $368 |
|  | // But vet doesn't know ADJSP, so suppress vet stack checking |
|  | NOP SP |
|  | MOVQ AX, 0(SP) |
|  | MOVQ CX, 8(SP) |
|  | MOVQ DX, 16(SP) |
|  | MOVQ BX, 24(SP) |
|  | MOVQ SI, 32(SP) |
|  | MOVQ DI, 40(SP) |
|  | MOVQ R8, 48(SP) |
|  | MOVQ R9, 56(SP) |
|  | MOVQ R10, 64(SP) |
|  | MOVQ R11, 72(SP) |
|  | MOVQ R12, 80(SP) |
|  | MOVQ R13, 88(SP) |
|  | MOVQ R14, 96(SP) |
|  | MOVQ R15, 104(SP) |
|  | #ifdef GOOS_darwin |
|  | #ifndef hasAVX |
|  | CMPB internal∕cpu·X86+const_offsetX86HasAVX(SB), $0 |
|  | JE 2(PC) |
|  | #endif |
|  | VZEROUPPER |
|  | #endif |
|  | MOVUPS X0, 112(SP) |
|  | MOVUPS X1, 128(SP) |
|  | MOVUPS X2, 144(SP) |
|  | MOVUPS X3, 160(SP) |
|  | MOVUPS X4, 176(SP) |
|  | MOVUPS X5, 192(SP) |
|  | MOVUPS X6, 208(SP) |
|  | MOVUPS X7, 224(SP) |
|  | MOVUPS X8, 240(SP) |
|  | MOVUPS X9, 256(SP) |
|  | MOVUPS X10, 272(SP) |
|  | MOVUPS X11, 288(SP) |
|  | MOVUPS X12, 304(SP) |
|  | MOVUPS X13, 320(SP) |
|  | MOVUPS X14, 336(SP) |
|  | MOVUPS X15, 352(SP) |
|  | CALL ·asyncPreempt2(SB) |
|  | MOVUPS 352(SP), X15 |
|  | MOVUPS 336(SP), X14 |
|  | MOVUPS 320(SP), X13 |
|  | MOVUPS 304(SP), X12 |
|  | MOVUPS 288(SP), X11 |
|  | MOVUPS 272(SP), X10 |
|  | MOVUPS 256(SP), X9 |
|  | MOVUPS 240(SP), X8 |
|  | MOVUPS 224(SP), X7 |
|  | MOVUPS 208(SP), X6 |
|  | MOVUPS 192(SP), X5 |
|  | MOVUPS 176(SP), X4 |
|  | MOVUPS 160(SP), X3 |
|  | MOVUPS 144(SP), X2 |
|  | MOVUPS 128(SP), X1 |
|  | MOVUPS 112(SP), X0 |
|  | MOVQ 104(SP), R15 |
|  | MOVQ 96(SP), R14 |
|  | MOVQ 88(SP), R13 |
|  | MOVQ 80(SP), R12 |
|  | MOVQ 72(SP), R11 |
|  | MOVQ 64(SP), R10 |
|  | MOVQ 56(SP), R9 |
|  | MOVQ 48(SP), R8 |
|  | MOVQ 40(SP), DI |
|  | MOVQ 32(SP), SI |
|  | MOVQ 24(SP), BX |
|  | MOVQ 16(SP), DX |
|  | MOVQ 8(SP), CX |
|  | MOVQ 0(SP), AX |
|  | ADJSP $-368 |
|  | POPFQ |
|  | POPQ BP |
|  | RET |


```

这段汇编代码很简单，把各种寄存器保存到栈中，然后调用 asyncPreempt2 函数，这个函数会恢复这些寋存器。



```


|  | func asyncPreempt2() { |
| --- | --- |
|  | gp := getg() |
|  | gp.asyncSafePoint = true |
|  | if gp.preemptStop { |
|  | mcall(preemptPark) |
|  | } else { |
|  | mcall(gopreempt_m) |
|  | } |
|  | gp.asyncSafePoint = false |
|  | } |


```

这个代码就是开始交给 g0 去执行调度任务，当 goroutine 回来可以继续执行的时候，会执行恢复寄存器的代码。


![](https://img2024.cnblogs.com/blog/2344773/202412/2344773-20241208173027354-320162724.png)


