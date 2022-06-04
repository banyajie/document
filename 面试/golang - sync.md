##### Go  Sync包详解

###### 乐观锁

> 实现思路是先让冲突产生，提交时再解决冲突。实现无锁操作

###### 悲观锁

> 实现思路是同一时刻只允许一个goroutine获取到锁,执行操作前必须先获取到锁，mysql的for update就是一种悲观锁.

###### 总线程锁定、MESI（缓存一致性协议）

> 总线锁定是指当CPU在处理某个数据时，通过锁定总线和主存的通信，让其他CPU不能访问主存的数据，从而达到独占CPU保证数据一致性的目的。总线锁定的开销很大，只有当CPU不支持MESI协议或者数据跨越多个缓存行才会降级为总线锁定
>
> 
>
> 当我们有一个变量,被CPU的多个core的高速缓冲区同时缓存了,这时我们就需要 通过MESI协议来保证该变量的读写的一致性.
>
> MESI表示4种状态:
>
> M(Modifyd) : 数据被修改了
>
> E(Exclusive): 独享数据
>
> S(Shard):共享数据
>
> I(Invalid):无效数据
>
> MESI数据状态转换过程：
>
> 当core1从主存读取变量x到core1的高速缓存区时,其他core都没有缓存这个变量地址,所以core1独享x，x的状态是E. 接着core2也读取了x变量到自己的高速缓存区,此时通过嗅探总线事件core1,core2会得知x不止一个副本.所以他们会把x的状态改为S.接着core1修改了x的值,core1把状态改为M,再通知core2把x状态改为无效I,当 core1确认所有缓存x的core都提交了I修改后把x的新值刷写到主存,再把x的状态改为独享E.之后core2需要再次读取x时,重新发起读指令从主存读取数据. 通过以上步骤，保障了多core操作同一数据 时的一致性.

###### Futex***(快速用户空间互斥锁)\***

> 用户态加锁的实现思路就是使用一个变量标示 0为无竞争,能获取锁. 大于0为有竞争,需要等待. 这里需要等待时,进入忙等,不停尝试获取锁直到成功(自旋锁).而不是休眠当前进程.这种用户空间的锁实现如果锁粒度较小能快速释放锁时,避免了程序休眠和唤醒,效率比较高.但是如果锁的粒度较大不能快速释放锁时,这种忙等就会导致程序阻塞.
>
> 内核态加锁. 虽然实现了进程的休眠和唤醒,但是不管有没有锁竞争每次lock和unlock都会陷入内核态.
>
> futex是一种用户态和内核态混用的同步机制.先在用户态检测是否有锁竞争,再决定要不要进入内核态休眠或唤醒进程. 这种机制减少了不必要的进入内核态.

###### PV

> PV操作是一种信号量机制,用于控制竞争资源的使用,分为P操作和V操作. P(s)操作为申请使用资源,V(s)操作为释放资源,是一个生产者和消费者的关系. 比如我们现在有某一个资源s为100,现在P(s)操作会先判断s是否大于0.如果大于0则把s减1,结束操作. 如果s<=0，则说明资源不充足了,需要阻塞等待资源. 然后是V(s)操作的时候把s+1,如果s<=0则释放等待队列中的第一个进程,否则s>0，则说明资源充足不需要等待.

###### sema（信号量）

> sema是golang实现的一个类似于Futex的信号量库.用于阻塞和唤醒 goroutine.
>
> ```go
> type semaRoot struct {
>     lock  mutex
>     //一棵平衡二叉树,存储阻塞在这个锁上的Goroutine.
>     treap *sudog
>     //阻塞在这个semaRoot上的goroutine数量
>     nwait uint32
> }
> const semTabSize = 251
> //addr会 hash到这个251元素的数组里面.
> var semtable [semTabSize]struct {
> 	// 阻塞的goroutine
> 	root semaRoot
> 	//cpu对齐操作.占位符
> 	pad  [cpu.CacheLinePadSize - unsafe.Sizeof(semaRoot{})]byte
> }
> ```
>
> 1. semacquire1() 加锁时调用,阻塞休眠当前goroutine.对于P(s)操作,申请资源
> 2. semrelease1()解锁时调用,唤醒goroutine,对应V(s)操作.释放资源
>
> ```go
> // P（S）操作,申请资源. 减信号量. 结果不为负数, P执行完毕.否则阻塞当前G,等待释放.
> func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags, skipframes int) {
>     if cansemacquire(addr) {
>         //其实这个方法就是判断addr是否大于0
>         //信号量为正数，说明资源还是充足的,完成操作,不需要休眠G
>         return
>     }
> 
>     // 执行到这里说明addr已经等于0了,已经没有足够的资源了,需要休眠goroutine
>     //获取一个sudog
>   
>     s := acquireSudog()
>     //按照addr hash到251其中一个bucket.而相同 addr 上的 sudog 会形成一个链表。
>     root := semroot(addr)
> 
>     for {
>         lockWithRank(&root.lock, lockRankRoot)
>         //增加等待数量,用于释放资源时快速判断.
>         atomic.Xadd(&root.nwait, 1)
>         // 再次检查资源数量是否足够(上次检查后执行到这里,资源数量可能被修改了).
>         if cansemacquire(addr) {
>             //资源数量为正数,减少等待数量,不休眠goroutine,返回
>             atomic.Xadd(&root.nwait, -1)
>             unlock(&root.lock)
>             break
>         }
>       
>       	// 加入到 semaRoot树上, lifo决定插入的位置.
>         root.queue(addr, s, lifo)
>         // 解锁semaRoot. 把当前g改为等待状态,让当前m调用其他g,当前g相当于等待.
>         goparkunlock(&root.lock, waitReasonSemacquire, traceEvGoBlockSync, 4+skipframes)
>         if s.ticket != 0 || cansemacquire(addr) {
>             break
>         }
>     }
> 
>     //调整P的sudog队列,如果P的sudog队列满了,则需要移动一半到全局sudog队列内去
>     releaseSudog(s)
> }
> 
> ```
>
> ```go
> //V（s）操作,释放资源, 加信号量.如果结果是负数,则释放一个因为P(s)而阻塞的goroutine.
> func semrelease1(addr *uint32, handoff bool, skipframes int) {
> 	root := semroot(addr)
> 	//信号量+1
> 	atomic.Xadd(addr, 1)
> 
> 	// 没有等待的sudog了,直接返回, 在申请资源时,nwait每次都会+1.
> 	if atomic.Load(&root.nwait) == 0 {
> 		return
> 	}
> 
> 	// 有等待的sudog,需要释放一个.
> 
> 	// 搜索等待列表
> 	lockWithRank(&root.lock, lockRankRoot)
> 	if atomic.Load(&root.nwait) == 0 {
> 		//再次检查，防止nwait在其他goroutine被修改
> 		unlock(&root.lock)
> 		return
> 	}
> 	//从等待队列取出一个sudog.
> 	s, t0 := root.dequeue(addr)
> 	if s != nil {
> 		//减少等待数量
> 		atomic.Xadd(&root.nwait, -1)
> 	}
> 	unlock(&root.lock)
> 	if s != nil {
> 		//如果是手递手释放(饥饿模式).释放队列首部的sudog
> 		if handoff && cansemacquire(addr) {
> 			s.ticket = 1
> 		}
> 		//把sudo对应的g改为待执行状态，并且放到P的本地队列下一个执行.
> 		readyWithTime(s, 5+skipframes)
> 		if s.ticket == 1 && getg().m.locks == 0 {
> 			// 释放队列首部的sudog
> 			// 调用调度器，立即执行P
> 			goyield()
> 		}
> 	}
> }
> 
> ```
>
> 

###### sync.cas

> CAS(Compare-and-Swap) ,比较交换,这是并发编程中常用到的一种技术
>
> ```go
> //golang  的cas大概实现了这么几类方法.
> atomic.AddInt64()
> atomic.LoadInt64()
> atomic.StoreInt64()
> atomic.SwapInt64()
> atomic.CompareAndSwapInt64()
> 
> ```
>
> 

###### sync.atomic

> ```go
> TEXT runtime∕internal∕atomic·Xadd64(SB), NOSPLIT, $0-24
> 	MOVQ	ptr+0(FP), BX
> 	MOVQ	delta+8(FP), AX
> 	MOVQ	AX, CX
>         //关键的LOCK指令.
> 	LOCK
> 	XADDQ	AX, 0(BX)
> 	ADDQ	CX, AX
> 	MOVQ	AX, ret+16(FP)
> 	RET
> 
> //交换比较
> atomic.CompareAndSwapInt64(addr *int64, old, new int64)
> //我们再看交换比较是怎么实现的.
> 
> TEXT runtime∕internal∕atomic·Cas64(SB), NOSPLIT, $0-25
> 	MOVQ	ptr+0(FP), BX
> 	MOVQ	old+8(FP), AX
> 	MOVQ	new+16(FP), CX
>         #还是LOCK指令
> 	LOCK
> 	CMPXCHGQ	CX, 0(BX)
> 	//取ZF标志寄存器的值，判断比较结果是否相等.
>         SETEQ	ret+24(FP)
> 	RET
> ```

###### sync.Map

> ```go
> 
> type Map struct {
> 	// 当涉及到脏数据(dirty)操作时候，需要使用这个锁
> 	mu Mutex
> 	// 只读数据，所以并发是安全的。实际存的是readOnly的数据结构。
> 	read atomic.Value // readOnly
> 	// 包含最新写入的数据。当misses计数达到一定值，将其赋值给read。
> 	dirty map[interface{}]*entry
> 	//计数作用。每次从read中读失败，则计数+1。到达一定的值后同步dirty和read
> 	misses int
> }
> 
> type readOnly struct {
> 	//真实的数据存放在这
> 	m       map[interface{}]*entry
> 	//如果是true,则说明read的数据和 dirty的数据不一致. 某些值不在read.但是在dirty内有。
> 	amended bool // true if the dirty map contains some key not in m.
> }
> 
> type entry struct {
> 	// 保存vaule的interface指针.
> 	// 如果 p == nil: 键值已经被删除，且 m.dirty == nil
> 	// p == expunged: 键值已经被删除，但 m.dirty!=nil 且 m.dirty 不存在该键值（expunged 实际是空接口指针）
> 	//除以上情况，则键值对存在，存在于 m.read.m 中，如果 m.dirty!=nil 则也存在于 m.dirty
> 	p unsafe.Pointer // *interface{}
> }
> 
> ```
>
> 

###### sync.Mutex

###### sync.RWMutex

###### sync.once

###### sync.pool

> ```go
> type Pool struct {
> 	noCopy noCopy
> 
> 	//固定大小per-P池, 实际类型为 [P]poolLocal
> 	local     unsafe.Pointer
> 	//local array 的大小
> 	localSize uintptr
> 
> 	//上一个周期的数据(local)
> 	victim     unsafe.Pointer 
> 	//上一个周期的数据大小(local array 的大小)
> 	victimSize uintptr
> 
> 	// 当get()失败时的返回值.不设置就会返回nil
> 	New func() interface{}
> }
> 
> //一个P持有一个poolLocal
> type poolLocalInternal struct {
> 	//本地P使用,不需要加锁,高速访问
> 	private interface{} 
> 	//所有P共享,需要加锁访问.使用一个链表存储
> 	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
> }
> 
> type poolLocal struct {
> 	poolLocalInternal
> 
> 	// 占位符,将 poolLocal 补齐至两个缓存行的倍数，防止 false sharing
> 	// false sharing: 多个变量共用一个cpu的高速缓存cache line.而导致一个变量更新,其他变量跟随失效的问题.
> 	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
> }
> ```

###### sync.WaitGroup

> ```go
> type WaitGroup struct {
> 	noCopy noCopy
> 	// 这里的waiter等待数指有几个地方调用了wait在阻塞中. 任务计数指调用的add()还没done()的数量
> 	//64位：state1[0] waiter等待计数器,state1[1]counter任务计数,state1[2]sema信号量,控制唤醒
> 	//32位: state1[0]sema信号量,state1[1]waiter,state1[2]counter
> 	state1 [3]uint32
> }
> 
> ```
>
> Add Done
>
> ```go
> //增加或减少任务计数 Done()调用Add()时,delta为负数
> func (wg *WaitGroup) Add(delta int) {
> 	//statep 两个计数器的值， semap信号量控制唤醒
> 	statep, semap := wg.state()
> 	state := atomic.AddUint64(statep, uint64(delta)<<32)
> 	v := int32(state >> 32)  //任务计数
> 	w := uint32(state)	//等待的数量
> 	//任务数量不能为负数(done()的时候不能超过Add()的数量)
> 	if v < 0 {
> 		panic("sync: negative WaitGroup counter")
> 	}
> 	//v == int32(delta) 表示这次调用add之前.任务数量是0个.之前没有任务数量.
> 	//已经有人调用了wait()在等待，现在又有人调用add()添加任务,并且之前任务数量等于0.重用的时候会出现这种状况.
> 	if w != 0 && delta > 0 && v == int32(delta) {
> 		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
> 	}
> 	//添加计数器成功. 一般add()传入的是正数, v会大于0.所以add()执行到这里就结束了.
> 	if v > 0 || w == 0 {
> 		return
> 	}
> 	//执行到这里.说明v==0 && w>0. 所有的add都已经done过了.接下来我们需要释放所有的wait.
> 	if *statep != state {
> 		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
> 	}
> 	//清空waiter数量
> 	*statep = 0
> 	for ; w != 0; w-- {
> 		//遍历释放所有wait.通知解除wait阻塞.
> 		runtime_Semrelease(semap, false, 0)
> 	}
> }
> 
> ```
>
> 

