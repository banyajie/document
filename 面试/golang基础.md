#### 数据结构和底层实现原理

###### string

>   源码
>
>   ```go
>   type stringStruct struct {
>       str unsafe.Pointer
>       len int
>   }
>   ```
>
>   tips：
>
>   -   string可以为空（长度为0），但不会是nil；
>   -   string对象不可以修改
>
>   字符串构建过程
>
>   ```go
>   // 先根据字符串构建stringStruct结构，再转换成string
>   
>   func gostringnocopy(str *byte) string { // 根据字符串地址构建string
>       ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)} // 先构造stringStruct
>       s := *(*string)(unsafe.Pointer(&ss))                             // 再将stringStruct转换成string
>       return s
>   }
>   ```
>
>   问题：
>
>   -   为什么string类型的数据不允 许修改？？
>       -   go语言的实现中，string结构不包含内存空间，只有一个内存的指针，这样的好处是string非常轻量，可以很方便的传递并且不会发生内存拷贝。
>       -   string通常指向字符串字面量，而字符串字面量的存储位置在只读段上，而不是堆栈。
>   -   []byte 转换string一定会发生内存转换吗？
>       -   临时场景下不会发生内存转换，如：字符串拼接（”<” + “string(b)” + “>”）、字符串比较（string(b) == “foo”）、
>   -   []byte、string如何选择？
>       -   是否需要切片操作？
>       -   是否需要nil值？
>       -   是否需要修改？

###### slice

>   源码
>
>   ```go
>   type slice struct {
>       array unsafe.Pointer
>       len   int
>       cap   int
>   }
>   
>   // make
>   tmp := make([]int, len, camp)   // 指定长度、容量
>   
>   // 使用数组创建
>   slice := array[5:7]
>   sliceA := make([]int, 5, 10)  //length = 5; capacity = 10
>   sliceB := sliceA[0:5]         //length = 5; capacity = 10
>   sliceC := sliceA[0:5:5]       //length = 5; capacity = 5
>   
>   // slice 扩容问题
>   // 使用append向Slice追加元素时，如果Slice空间不足，将会触发Slice扩容，扩容实际上是重新分配一块更大的内存，将原Slice数据拷贝进新Slice，然后返回新Slice，扩容后再将数据追加进去
>   
>   // 如果原Slice容量小于1024，则新Slice容量将扩大为原来的2倍；
>   // 如果原Slice容量大于等于1024，则新Slice容量将扩大为原来的1.25倍；
>   
>   ```
>
>   问题：
>
>   -   slice copy
>       -   使用copy()内置函数拷贝两个切片时，会将源切片的数据逐个拷贝到目的切片指向的数组中，拷贝数量取两个切片长度的最小值。
>       -   比如：长度为10的切片拷贝到长度为5的切片时，将会拷贝5个元素
>       -   所以：slice copy不会发生扩容
>   -   tips
>       -   创建切片，可以预分配容量。减少扩容的发生
>       -   slice copy时注意生效长度
>       -   谨慎使用多个切片操作同一个数组。防止发生读写冲突

###### map

>   hash表，一个哈希表里可以有多个哈希表节点，也即bucket，而每个bucket就保存了map中的一个或一组键值对。
>
>   ```go
>   type hmap struct {
>       count     int // 当前保存的元素个数
>       ...
>       B         uint8
>       ...
>       buckets    unsafe.Pointer // bucket数组指针，数组的大小为2^B
>       ...
>   }
>   
>   // bucket 数据结构
>   type bmap struct {
>       tophash [8]uint8 // 存储哈希值的高8位
>       data    byte[1]  // key value数据:key/key/key/.../value/value/value...
>       overflow *bmap   // 溢出bucket的地址（hash冲突的链地址）
>   }
>   
>   // tophash是个长度为8的数组，哈希值相同的键（准确的说是哈希值低位相同的键）存入当前bucket时会将哈希值的高位存储在该数组中，以方便后续匹配。
>   // data区存放的是key-value数据，存放顺序是key/key/key/…value/value/value，如此存放是为了节省字节对齐带来的空间浪费。
>   // overflow 指针指向的是下一个bucket，据此将所有冲突的键连接起来。
>   
>   
>   ```
>
>   -   负载因子
>       -   负载因子 = 键数量 / bucket数量
>       -   哈希因子过小，说明空间利用率低
>       -   哈希因子过大，说明冲突严重，存取效率低
>   -   扩缩容
>       -   负载因子 > 6.5 时扩容，也即平均每个bucket存储的键值对达到6.5个
>       -   overflow数量 > 2^15时，也即overflow数量超过32768时
>       -   增量渐进式扩容（Go采用逐步搬迁策略，即每次访问map时都会触发一次搬迁，每次搬迁2个键值对）
>       -   等量扩容：实际上并不是扩大容量，buckets数量不变，重新做一遍类似增量扩容的搬迁动作，把松散的键值对重新排列一次，以使bucket的使用率更高，进而保证更快的存取。极端场景下，比如不断地增删，而键值对正好集中在一小部分的bucket，这样会造成overflow的bucket数量增多，但负载因子又不高，从而无法执行增量搬迁的情况
>   -   查找过程
>       -   根据key计算hash值
>       -   根据hash值的低位与hmap.B取模确定bucket在数组指针中的位置
>       -   取hash值高八位在tophash中查找，如果tophash[i]中存储值与哈希值高位相等，则去找到该bucket中的key值进行比较，当前bucket没有找到，则继续从下个overflow的bucket中查找
>       -   如果当前处于搬迁过程，则优先从oldbuckets查找
>       -   如果找不到，不会返回nil
>   -   插入过程
>       -   key计算hash
>       -   查找bucket位置
>       -   查找该key是否已经存在，如果存在则直接更新值，不存在插入

###### chan

>   源码
>
>   ```go
>   type hchan struct {
>       qcount   uint           // 当前队列中剩余元素个数
>       dataqsiz uint           // 环形队列长度，即可以存放的元素个数
>       buf      unsafe.Pointer // 环形队列指针
>       
>       elemsize uint16         // 每个元素的大小
>       closed   uint32         // 标识关闭状态
>       elemtype *_type         // 元素类型
>       
>       sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
>       recvx    uint           // 队列下标，指示元素从队列的该位置读出
>       
>       recvq    waitq          // 等待读消息的goroutine队列
>       sendq    waitq          // 等待写消息的goroutine队列
>       
>       lock mutex              // 互斥锁，chan不允许并发读写
>   }
>   
>   // make chan
>   tmp := make(chan, int, 10)
>   
>   func makechan(t *chantype, size int) *hchan {
>       var c *hchan
>       c = new(hchan)
>       c.buf = malloc(元素类型大小*size)
>       c.elemsize = 元素类型大小
>       c.elemtype = 元素类型
>       c.dataqsiz = size
>   
>       return c
>   }
>   
>   ```
>
>   常见用法
>
>   -   单向channel
>
>       -   ```go
>           // 实际上没有单向channel
>           // 所谓单向channel只是对channel的一种使用限制
>           // func readChan(chanName <-chan int)： 通过形参限定函数内部只能从channel中读取数据
>           // func writeChan(chanName chan<- int)： 通过形参限定函数内部只能向channel中写入数据
>                           
>           func readChan(chanName <-chan int) {
>               <- chanName
>           }
>                           
>           func writeChan(chanName chan<- int) {
>               chanName <- 1
>           }
>           ```
>
>   -   select
>
>       -   ```go
>           for {
>               select {
>               case e := <- chan1:
>                   println("")
>               case e := <- chan2:
>                   println("")
>               default:
>                   println("")
>               }
>           }
>                           
>           // 如果channel中没有可读数据，select有default的情况下是不会阻塞的
>                           
>           ```
>
>   -   range
>
>       -   ```go
>           func chanRange(chanName chan int) {
>               for e := range chanName {
>                   fmt.Printf("Get element from chan: %d\n", e)
>               }
>           }
>                                           
>           //注意：如果向此channel写数据的goroutine退出时，系统检测到这种情况后会panic，否则range将会永久阻塞。
>                                           
>           ```
>
>           

###### struct

>    Struct
>
>   ```go
>   // A StructField describes a single field in a struct.
>   type StructField struct {
>       // Name is the field name.
>       Name string
>       ...
>       Type      Type      // field type
>       Tag       StructTag // field tag string
>       ...
>   }
>   
>   type StructTag string
>   ```
>
>   Tag
>
>   -   读取tag
>
>       -   ```go
>           package main
>                                       
>           import (
>               "reflect"
>               "fmt"
>           )
>                                       
>           type Server struct {
>               ServerName string `key1:"value1" key11:"value11"`
>               ServerIP   string `key2:"value2"`
>           }
>                                       
>           func main() {
>               s := Server{}
>               st := reflect.TypeOf(s)
>                                       
>               field1 := st.Field(0)
>               fmt.Printf("key1:%v\n", field1.Tag.Get("key1"))
>               fmt.Printf("key11:%v\n", field1.Tag.Get("key11"))
>                                       
>               filed2 := st.Field(1)
>               fmt.Printf("key2:%v\n", filed2.Tag.Get("key2"))
>           }
>           ```
>
>       -   

###### iota

>   const 表达式
>
>   ```go
>   type Priority int
>   const (
>       LOG_EMERG Priority = iota
>       LOG_ALERT
>       LOG_CRIT
>       LOG_ERR
>       LOG_WARNING
>       LOG_NOTICE
>       LOG_INFO
>       LOG_DEBUG
>   )
>   // iota = 0 ==> LOG_EMERG = 0, 后面自增1
>   
>   const (
>       mutexLocked = 1 << iota // mutex is locked
>       mutexWoken
>       mutexStarving
>       mutexWaiterShift = iota
>       starvationThresholdNs = 1e6
>   )
>   // mutexLocked = 1 mutexWoken = 2 mutexStarving = 4
>   // mutexWaiterShift = 3 starvationThresholdNs = 1000000
>   
>   const (
>       bit0, mask0 = 1 << iota, 1<<iota - 1
>       bit1, mask1
>       _, _          // 
>       bit3, mask3
>   )
>   // bit0 == 1， mask0 == 0 
>   // bit1 == 2， mask1 == 1 
>   // bit3 == 8， mask3 == 7  
>   ```
>
>   规则：
>
>   -   iota代表了const声明块的行索引（下标从0开始）
>   -   const声明还有个特点，即第一个常量必须指定一个表达式，后续的常量如果没有表达式，则继承上面的表达式。
>
>   编译原理：
>
>   ```go
>   // const中每一行的结构	
>   // A ValueSpec node represents a constant or variable declaration
>   // (ConstSpec or VarSpec production).
>   //
>   ValueSpec struct {
>       Doc     *CommentGroup // associated documentation; or nil
>       Names   []*Ident      // value names (len(Names) > 0)
>       Type    Expr          // value type; or nil
>       Values  []Expr        // initial values; or nil
>       Comment *CommentGroup // line comments; or nil
>   }
>   
>   // ValueSpec.Names， 这个切片中保存了一行中定义的常量，如果一行定义N个常量，那么ValueSpec.Names切片长度即为N。
>   // const块实际上是spec类型的切片，用于表示const中的多行。
>   
>   // 编译逻辑
>   for iota, spec := range ValueSpecs {
>       for i, name := range spec.Names {
>           obj := NewConst(name, iota...) //此处将iota传入，用于构造常量
>           ...
>       }
>   }
>   
>   ```
>
>   

#### 控制结构和底层实现原理

###### defer

>   源码结构
>
>   ```go
>   type _defer struct {
>       sp      uintptr   // 函数栈指针
>       pc      uintptr   // 程序计数器
>       fn      *funcval  // 函数地址
>       link    *_defer   // 指向自身结构的指针，用于链接多个defer，每次新增defer从头部插入，执行时从头部开始执行
>   }
>   ```
>
>   规则：
>
>   -   延迟函数的参数在defer语句出现时就确定了
>   -   执行顺序：后劲先出
>   -   延迟函数可能操作主函数的具名返回值
>       -   返回常量？
>       -   匿名返回值，返回变量？
>       -   具名返回值返回？
>
>   Recover
>
>   ```go
>   // 使用方式
>   func NoPanic() {
>       if err := recover(); err != nil {
>           fmt.Println("Recover success...")
>       }
>   }
>   
>   func Dived(n int) {
>       defer NoPanic()
>   
>       fmt.Println(1/n)
>   }
>   ```
>
>   -   recover失效
>       -   panic时指定的参数为`nil`；（一般panic语句如`panic("xxx failed...")`）
>       -   当前协程没有发生panic
>       -   recover没有被defer方法直接调用

###### select

>   ```go
>   type scase struct {
>       c           *hchan         // 当前case语句所操作的channel指针，这也说明了一个case语句只能操作一个channel
>       kind        uint16		   		// 该case的类型，分为读channel、写channel和default，常量定义：caseRev、caseSend、caseDefault
>       elem        unsafe.Pointer // 缓冲区地址
>   }
>   
>   // cas0为scase数组的首地址，selectgo()就是从这些scase中找出一个返回。
>   // order0为一个两倍cas0数组长度的buffer，保存scase随机序列pollorder和scase中channel地址序列lockorder
>   // 		pollorder：每次selectgo执行都会把scase序列打乱，以达到随机检测case的目的。
>   // 		lockorder：所有case语句中channel序列，以达到去重防止对channel加锁时重复加锁的目的。
>   // ncases表示scase数组的长度
>   func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
>       //1. 锁定scase语句中所有的channel
>       //2. 按照随机顺序检测scase中的channel是否ready
>       //   2.1 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index, true)
>       //   2.2 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index, false)
>       //   2.3 所有case都未ready，则解锁所有的channel，然后返回（default index, false）
>       //3. 所有case都未ready，且没有default语句
>       //   3.1 将当前协程加入到所有channel的等待队列
>       //   3.2 当将协程转入阻塞，等待被唤醒
>       //4. 唤醒后返回channel对应的case index
>       //   4.1 如果是读操作，解锁所有的channel，然后返回(case index, true)
>       //   4.2 如果是写操作，解锁所有的channel，然后返回(case index, false)
>   }
>   
>   ```
>
>   总结：
>
>   -   select语句中除default外，每个case操作一个channel，要么读要么写
>   -   select语句中除default外，各case执行顺序是随机的
>   -   select语句中如果没有default语句，则会阻塞等待任一case
>   -   select语句中读操作要判断是否成功读取，关闭的channel也可以读取（case elem, ok := <-chan1:）

###### range

>   可以操作数组、slice、map、chan，底层实现并不相同
>
>   ```go
>   // 编译过程中根据操作数据类型生成对应代码
>   
>   // Arrange to do a loop appropriate for the type.  We will produce
>   //   for INIT ; COND ; POST {
>   //           ITER_INIT
>   //           INDEX = INDEX_TEMP
>   //           VALUE = VALUE_TEMP // If there is a value
>   //           original statements
>   //   }
>   
>   // slice、数组
>   // The loop we generate:
>   //   for_temp := range
>   //   len_temp := len(for_temp)
>   //   for index_temp = 0; index_temp < len_temp; index_temp++ {
>   //           value_temp = for_temp[index_temp]
>   //           index = index_temp
>   //           value = value_temp
>   //           original body
>   //   }
>   // 遍历slice的时候，首先拿slice长度作为循环次数，循环体内先获取元素值，如果for-range中接收index和value的话，则会对index和value进行一次赋值
>   
>   // map
>   // The loop we generate:
>   //   var hiter map_iteration_struct
>   //   for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
>   //           index_temp = *hiter.key
>   //           value_temp = *hiter.val
>   //           index = index_temp
>   //           value = value_temp
>   //           original body
>   //   }
>   // 遍历map时没有指定循环次数，循环体与遍历slice类似。由于map底层实现与slice不同，map底层使用hash表实现，插入数据位置是随机的，所以遍历过程中新插入的数据不能保证遍历到
>   
>   
>   // channel
>   // The loop we generate:
>   //   for {
>   //           index_temp, ok_temp = <-range
>   //           if !ok_temp {
>   //                   break
>   //           }
>   //           index = index_temp
>   //           original body
>   //   }
>   // channel遍历是依次从channel中读取数据,读取前是不知道里面有多少个元素的。如果channel中没有元素，则会阻塞等待，如果channel已被关闭，则会解除阻塞并退出循环。
>   
>   
>   
>   ```
>

#### 锁

###### mutex

>   互斥锁
>
>   ```go
>   type Mutex struct {
>      state int32			// 互斥锁状态
>      sema  uint32		  // 信号量，协程阻塞等待该信号量，解锁的协程释放信号量从而唤醒等待信号量的协程
>   }
>   
>   const (
>      mutexLocked = 1 << iota // mutex is locked
>      mutexWoken
>      mutexStarving
>      mutexWaiterShift = iota
>      starvationThresholdNs = 1e6
>   )
>   // mutexLocked = 1 
>   // mutexWoken = 2 
>   // mutexStarving = 4
>   // mutexWaiterShift = 3 
>   // starvationThresholdNs = 1000000
>   
>   // Mutex的四种状态
>   // Locked: 表示该Mutex是否已被锁定，0：没有锁定 1：已被锁定。
>   // Woken: 表示是否有协程已被唤醒，0：没有协程唤醒 1：已有协程唤醒，正在加锁过程中。
>   // Starving：表示该Mutex是否处于饥饿状态，0：没有饥饿 1：饥饿状态，说明有协程阻塞了超过1ms。
>   // Waiter: 表示阻塞等待锁的协程个数，协程解锁时根据此值来判断是否需要释放信号量。
>   
>   两种模式：
>   normal、starvation
>   
>   woken状态
>   Woken状态用于加锁和解锁过程的通信，举个例子，同一时刻，两个协程一个在加锁，一个在解锁，在加锁的协程可能在自旋过程中，此时把Woken标记为1，用于通知解锁协程不必释放信号量了，好比在说：你只管解锁好了，不必释放信号量，我马上就拿到锁了。
>   
>   ```
>   
>   加锁过程
>
>   -   普通加锁成功    Locked = 1
>-   加锁阻塞            Locked = 1、Waiter += 1
>   
>   解锁
>
>   -   如果Waiter > 0 解锁后并唤醒协程，唤醒后获得锁
>
>   
>
>   自旋过程
>
>   加锁时，如果当前Locked位为1，说明该锁当前由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续的探测Locked位是否变为0，这个过程即为自旋过程
>
>   自旋时间很短，但如果在自旋过程中发现锁已被释放，那么协程可以立即获取锁。此时即便有协程被唤醒也无法获取锁，只能再次阻塞。
>
>   自旋的好处是，当加锁失败时不必立即转入阻塞，有一定机会获取到锁，这样可以避免协程的切换。
>
>   
>
>   自旋的原理？
>
>   自旋对应于CPU的”PAUSE”指令，CPU对该指令什么都不做，相当于CPU空转，对程序而言相当于sleep了一小段时间，时间非常短，当前实现是30个时钟周期
>
>   自旋过程中会持续探测Locked是否变为0，连续两次探测间隔就是执行这些PAUSE指令，它不同于sleep，不需要将协程转为睡眠状态。
>
>   
>
>   自旋条件？
>
>   -   自旋次数要足够小，通常为4，即自旋最多4次
>-   CPU核数要大于1，否则自旋没有意义，因为此时不可能有其他协程释放锁
>   -   协程调度机制中的Process数量要大于1，比如使用GOMAXPROCS()将处理器设置为1就不能启用自旋
>   -   协程调度机制中的可运行队列必须为空，否则会延迟协程调度
>   
>   
>
>   自旋的好处
>
>   充分的利用CPU，尽量避免协程切换。因为当前申请加锁的协程拥有CPU，如果经过短时间的自旋可以获得锁，当前协程可以继续运行，不必进入阻塞状态
>
>   
>
>   自旋的缺点
>
>   如果自旋过程中获得锁，那么之前被阻塞的协程将无法获得锁，如果加锁的协程特别多，每次都通过自旋获得锁，那么之前被阻塞的进程将很难获得锁，从而进入饥饿状态
>
>   为了避免协程长时间无法获取锁，自1.8版本以来增加了一个状态，即Mutex的Starving状态。这个状态下不会自旋，一旦有协程释放锁，那么一定会唤醒一个协程并成功加锁。
>



###### rwmutex

>   
>
>   ```go
>   type RWMutex struct {
>       w           Mutex  // 用于控制多个写锁，获得写锁首先要获取该锁，如果有一个写锁在进行，那么再到来的写锁将会阻塞于此
>       writerSem   uint32 // 写阻塞等待的信号量，最后一个读者释放锁时会释放信号量
>       readerSem   uint32 // 读阻塞的协程等待的信号量，持有写锁的协程释放锁后会释放信号量
>       readerCount int32  // 记录读者个数
>       readerWait  int32  // 记录写阻塞时读者个数
>   }
>   
>   // RLock()：读锁定
>   // RUnlock()：解除读锁定
>   // Lock(): 写锁定，与Mutex完全一致
>   // Unlock()：解除写锁定，与Mutex完全一致
>   
>   ```
>
>   -   Lock 逻辑
>       1.   获取Mutex（RWMutex.w.lock()），如果多个写锁排队
>       2.   readerCount > 0 则阻塞等待所有的读结束
>       3.   lock成功
>   -   Unlock逻辑
>       1.   唤醒因读锁定而被阻塞的协程（如果有的话）
>       2.   释放互斥锁
>   -   RLock逻辑
>       1.   增加读操作计数，即readerCount++
>       2.   阻塞等待写操作结束(如果有的话)
>   -   RUnlock逻辑
>       1.   减少读操作计数，即readerCount–
>       2.   唤醒等待写操作的协程（如果有的话）（最后一个读锁释放才唤醒阻塞写操作的协程）
>
>   问题：
>
>   -   写操作如何阻塞写操作？
>
>       -   互斥锁
>
>   -   写操作如何阻塞读操作？
>
>       -   RWMutex.readerCount是个整型值，用于表示读者数量，不考虑写操作的情况下，每次读锁定将该值+1，每次解除读锁定将该值-1，所以readerCount取值为[0, N]，N为读者个数，实际上最大可支持2^30个并发读者
>       -   当写锁定进行时，会先将readerCount减去2^30，从而readerCount变成了负值，此时再有读锁定到来时检测到readerCount为负值，便知道有写操作在进行，只好阻塞等待。而真实的读操作个数并不会丢失，只需要将readerCount加上2^30即可获得
>
>   -   读操作如何阻塞写操作？
>
>       -   读锁定会先将RWMutext.readerCount加1，此时写操作到来时发现读者数量不为0，会阻塞等待所有读操作结束。
>
>           所以，读操作通过readerCount来将来阻止写操作的。
>
>   -   为什么写锁定不会被饿死？
>
>       -   写操作要等待读操作结束后才可以获得锁，写操作等待期间可能还有新的读操作持续到来，如果写操作等待所有读操作结束，很可能被饿死。然而，通过RWMutex.readerWait可完美解决这个问题
>       -   写操作到来时，会把RWMutex.readerCount值拷贝到RWMutex.readerWait中，用于标记排在写操作前面的读者个数
>       -   前面的读操作结束后，除了会递减RWMutex.readerCount，还会递减RWMutex.readerWait值，当RWMutex.readerWait值变为0时唤醒写操作。
>       -   所以，写操作就相当于把一段连续的读操作划分成两部分，前面的读操作结束后唤醒写操作，写操作结束后唤醒后面的读操作

###### 信号量

> 同步问题的本质？
>
> 可以通过锁来保护临界区操作的互斥性
>
> 协程如何等待、唤醒？
>
> - runtime.semaphore   信号量（可供协程使用的信号量）
> - ![image-20220521154235701](/Users/banyajie/Library/Application Support/typora-user-images/image-20220521154235701.png)
> - runtime内部通过一个大小为251的semaTable来管理所有的semaphore。
> - 怎么通过一个固定大小的table来管理执行阶段数量不定的semaphore呢？
> - semaTable每个节点存储 的是平衡树的root 节点。平衡树中每个节点都是一个sudog对象
> - ![image-20220521154541890](/Users/banyajie/Library/Application Support/typora-user-images/image-20220521154541890.png)

#### 协程

###### 线程池缺陷

>   

###### 协程调度过程

>    GMP模型
>
>   ![image-20211123170159855](/Users/banyajie/Library/Application%20Support/typora-user-images/image-20211123170159855.png)
>
>   -   队列轮转
>   -   系统调用过程
>   -   工作量窃取提高效率

#### 内存管理

###### 内存分配原理

>   ![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_8aa79b32715f0ebe6e9618edd9d98383_r.png)
>
>   -   spans：存放span的指针，每个指针对应一个或多个page，所以span区域的大小为(512GB/8KB)*指针大小8byte = 512M
>   -   bitmap：用于GC
>   -   arena：堆区。大小为512G，为了方便管理把arena区域划分成一个个的page，每个page为8KB,一共有512GB/8KB个页
>
>   Span：
>
>   ```go
>   // 为了满足小对象分配，span中的一页会划分更小的粒度，而对于大对象比如超过页大小，则通过多页实现
>   
>   // class 对象
>   // 根据对象大小，划分了一系列class，每个class都代表一个固定大小的对象，以及每个span的大小
>   // class  bytes/obj  bytes/span  objects  waste bytes
>   // class  bytes/obj  bytes/span  objects  waste bytes
>   //     1          8        8192     1024            0
>   //     2         16        8192      512            0
>   //     3         32        8192      256            0
>   //     4         48        8192      170           32
>   //     5         64        8192      128            0
>   //     6         80        8192      102           32
>   //     7         96        8192       85           32
>   //     8        112        8192       73           16
>   //     9        128        8192       64            0
>   //    10        144        8192       56          128
>   
>   // class： class ID，每个span结构中都有一个class ID, 表示该span可处理的对象类型
>   // bytes/obj：该class代表对象的字节数
>   // bytes/span：每个span占用堆的字节数，也即页数*页大小
>   // objects: 每个span可分配的对象个数，也即（bytes/spans）/（bytes/obj）
>   // waste bytes: 每个span产生的内存碎片，也即（bytes/spans）%（bytes/obj）
>   
>   // 内存管理的基本单位，每个span用于管理特定的 class 对象，根据对象大小，span将一个或多个页拆分成多个块进行管理
>   type mspan struct {
>       next *mspan            //链表前向指针，用于将span链接起来
>       prev *mspan            //链表前向指针，用于将span链接起来
>       
>       startAddr uintptr // 起始地址，也即所管理页的地址
>       npages    uintptr // 管理的页数
>   
>       nelems uintptr // 块个数，也即有多少个块可供分配
>   
>       allocBits  *gcBits //分配位图，每一位代表一个块是否已分配
>   
>       allocCount  uint16     // 已分配块的个数
>       spanclass   spanClass  // class表中的class ID
>   
>       elemsize    uintptr    // class表中的对象大小，也即块大小
>   }
>   
>   
>   // 管理span
>   // 有了管理内存的基本单位span，还要有个数据结构来管理span，这个数据结构叫mcentral，各线程需要内存时从mcentral管理的span中申请内存，为了避免多线程申请内存时不断地加锁，
>   // Golang为每个线程分配了span的缓存，这个缓存即是cache
>   
>   type mcache struct {
>       alloc [67*2]*mspan // 按class分组的mspan列表
>   }
>   
>   // alloc为mspan的指针数组，数组大小为class总数的2倍。
>   // 数组中每个元素代表了一种class类型的span列表，每种class类型都有两组span列表，第一组列表中所表示的对象中包含了指针，第二组列表中所表示的对象不含有指针，这么做是为了提高GC扫描性能，对于不包含指针的span列表，没必要去扫描。
>   
>   // 根据对象是否包含指针，将对象分为noscan和scan两类，其中noscan代表没有指针，而scan则代表有指针，需要GC进行扫描。
>   
>   // mcache在初始化时是没有任何span的，在使用过程中会动态地从central中获取并缓存下来，根据使用情况，每种class的span个数也不相同。上图所示，class 0的span数比class1的要多，说明本线程中分配的小对象要多一些。
>   
>   
>   
>   // 全局资源 central
>   // 每个mcentral对象只管理特定的class规格的span。事实上每种class都会对应一个mcentral,这个mcentral的集合存放于mheap数据结构中
>   type mcentral struct {
>       lock      mutex     //互斥锁
>       spanclass spanClass // span class ID
>       nonempty  mSpanList // non-empty 指还有空闲块的span列表
>       empty     mSpanList // 指没有空闲块的span列表
>   
>       nmalloc uint64      // 已累计分配的对象个数
>   }
>   
>   // 线程从central获取span步骤如下：
>   // 加锁
>   // 从nonempty列表获取一个可用span，并将其从链表中删除
>   // 将取出的span放入empty链表
>   // 将span返回给线程
>   // 解锁
>   // 线程将该span缓存进cache
>   
>   // 线程将span归还步骤如下：
>   // 加锁
>   // 将span从empty列表删除
>   // 将span加入noneempty列表
>   // 解锁
>   
>   
>   // heap 堆
>   type mheap struct {
>       lock      mutex  // lock： 互斥锁
>   
>       spans []*mspan   // 指向spans区域，用于映射span和page的关系
>   
>       bitmap        uintptr     //指向bitmap首地址，bitmap是从高地址向低地址增长的
>   
>       arena_start uintptr        //指示arena区首地址
>       arena_used  uintptr        //指示arena区已使用地址位置
>   
>       central [67*2]struct {      // 每种class对应的两个mcentral
>           mcentral mcentral
>           pad      [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
>       }
>   }
>   
>   
>   ```
>
>   Mcache 和 mspan的对应关系
>
>   ![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_27075118cbd0843863bf715a0582c27c_r.png)
>
>   mheap内存管理示意图如下：
>
>   ![img](https://www.topgoer.cn/uploads/gozhuanjia/images/m_28744576778def4f0cefa8734bc82f59_r.png)
>

###### 堆内存管理

> 

#### GC

###### GC原理

>1.   进程的虚拟地址空间
>     1.   bss段：静态内存分配 - 可读写，常是指用来存放程序中未初始化的全局变量和静态变量的一块内存区域。
>     2.   代码段：程序要执行的指令
>     3.   （data段）数据段：全局变量。通常是指用来存放程序中已初始化的全局变量的一块内存区域。
>     4.   stack - 栈（函数栈针）：函数的局部变量、参数、返回值。
>          - 栈上的数据会随着函数调用的结束自动回收
>     5.   heap - 堆：堆上数据需要主动释放才可以被重新使用
>     6.   
>
>2.   常见的垃圾回收算法
>     -   引用计数
>         -   
>     -   标记、清除
>         -   缺点
>             -   内存碎片化
>             -   STW问题：暂停用户程序（用户程序可以暂停吗？）
>     -   分代收集
>3.   GC过程中root对象有哪些？
>     - bss段、数据段、协程栈上的root节点
>     - 从root节点去追踪到堆上
>     - 如何确定这些节点是否需要扫描？（Is Pointer）
>       - 编译阶段：go语言会在编译阶段会生成bss段、数据段对应的元数据，存储在可执行文件中。通过各个模块对应的moduledata可以获得 gcdatamask、gcbssmask信息。用于判断特定的root节点是否是指针
>       - 协程栈也有对应的元数据，存储在stackMap中。可以知道栈上的局部变量、参数、返回值等对象中哪些是存活的指针，确定了哪些节点是指针，还有判断这些指针是否执行了堆内存，如果指向了堆内存就加入到gc扫描的池子中
>       - 堆上数据也有元数据。mheap管理堆上一大段连续的内存
>4.   核心思想
>     - 目前自动垃圾回收算法都是使用“可达性”或者“存活性”的，要识别存活对象，可以把栈、数据段上的数据对象作为root。基于root进一步追踪。把可以追踪到的数据标记为存活对象，剩下的追踪不到的就是垃圾
>5.   三色抽象
>     - 三色抽象可以展现追踪式回收过程中对象状态的变化过程
>     - 1：垃圾回收开始时所有对象都为白色
>     - 2：把直接追踪到的root 节点标记为灰色，灰色表示根据当前节点展开的追踪还未完成。当灰色节点追踪完成后，会把灰色节点置位黑色节点
>     - 3：继续追踪灰色节点。直到没有灰色节点
>     - 4：剩余的白色节点就是垃圾
>6.   STW 问题
>     - 分多次Stop。用户程序和垃圾回收交替执行 - 增量式垃圾回收
>     - 
>7.   golang GC过程
>     - go语言采用的是标记清理算法，支持主体并发增量式回收，使用插入写屏障和删除写屏障的混合写屏障保证对象的准确回收
>     - 执行过程
>       - 准备阶段（Mark Setup）：为每一个 P （GMP模型中的processor）创建一个Mark Worker 协程，把对应的 g 指针存储到  P （p.gcBgMarkWork）中。这些后台Mark Workder 协程创建后很快进入休眠，等到标记阶段得到调度才执行
>       - GC Mark 阶段：第一次 Stop The World，进入Mark阶段，全局变量 gcphase记录gc阶段标识，全局变量 writeBarrier.Enable 记录是否开启写屏障，全局变量 gcBlackenEnabled = 1 标识是否进行 gc 标记工作
>       - Start The world：所有的P都会知道写屏障已经开启，然后 Mark Wroker协程就可以得到调度执行展开标记工作
>       - 第二次 Stop the world：GC 进入GCMarkTermination阶段，确认标记工作是否完成，然后停止标记工作（ gcBlackenEnabled = 0），gcphase = GCOFF阶段关闭写屏障，进入gcOff阶段之前。新分配的对象会直接标记为黑色，进入gcOff阶段之后。再分配的对象就是为白色了。
>       - Start the world：清扫阶段。执行清扫阶段的协程由：runtime.Main 在 gcenable 中创建。对应的g指针在全局变量sweep中。清扫阶段这个后台的sweep会被加入到 run queue中，得到调度执行时就会清扫数据
>8.   错误回收问题
>     - 强三色不变式：不允许有白色对象直接被黑色对象引用
>       - 核心 - 关注黑色对象对白色对象的引用行为
>       - 
>     - 弱三色不变式：如果有黑色对象引用白色对象，那必须保证白色对象可以通过灰色对象追踪到
>       - 核心 - 关注哪些对白色对象路径的破坏行为
>       - 比如：删除灰色对象到白色对象的引用时，可以把白色对象置位灰色 （删除写屏障）
>     - 实现强弱三色不变式
>       - 读写屏障
>       - 写屏障
>         - 会在写操作中插入指令，目的是把数据对象的修改通知垃圾回收器。所以，写屏障都会有个记录集，记录集通常采用顺序存储、哈希表记录精确到被修改的对象还是只记录其所在页
>         - 插入写屏障保证 - 强三色不变式：不允许黑色对象到白色对象的引用。 
>           - 可以把白色对象直接设置为灰色
>           - 把写入的黑色对象退到灰色
>         - 删除写屏障
>       - 读屏障
>         - 复制式垃圾回收情况下。保证对象有副本的情况下的引用问题
>
>
>
>

###### 逃逸分析

>1.   内存逃逸
>
>     >   所谓逃逸分析（Escape analysis）是指由编译器决定内存分配的位置，不需要程序员指定。
>     >   函数中申请一个新的对象
>     >
>     >   -   如果分配在栈中，则函数执行结束可自动将内存回收；
>     >   -   如果分配在堆中，则函数执行结束可交给GC（垃圾回收）处理；
>     >
>     >   
>
>2.   场景
>
>     >   -   指针逃逸
>     >   -   栈内存不足逃逸
>     >   -   动态类型逃逸（编译期间无法确定类型的）
>     >   -   闭包引用的对象

#### 并发控制

###### channel

>   

###### WaitGroup

>    源码
>
>   ```go
>   type WaitGroup struct {
>       state1 [3]uint32
>   }
>   
>   // state1是个长度为3的数组，其中包含了state和一个信号量，而state实际上是两个计数器：
>   
>   // counter： 当前还未执行结束的goroutine计数器
>   // waiter count: 等待goroutine-group结束的goroutine数量，即有多少个等候者
>   // semaphore: 信号量
>   
>   
>   // Add()做了两件事，一是把delta值累加到counter中，因为delta可以为负值，也就是说counter有可能变成0或负值，所以第二件事就是当counter值变为0时，根据waiter数值释放等量的信号量，把等待的goroutine全部唤醒，如果counter变为负值，则panic.
>   
>   func (wg *WaitGroup) Add(delta int) {
>       statep, semap := wg.state() //获取state和semaphore地址指针
>   
>       state := atomic.AddUint64(statep, uint64(delta)<<32) //把delta左移32位累加到state，即累加到counter中
>       v := int32(state >> 32) //获取counter值
>       w := uint32(state)      //获取waiter值
>   
>       if v < 0 {              //经过累加后counter值变为负值，panic
>           panic("sync: negative WaitGroup counter")
>       }
>   
>       //经过累加后，此时，counter >= 0
>       //如果counter为正，说明不需要释放信号量，直接退出
>       //如果waiter为零，说明没有等待者，也不需要释放信号量，直接退出
>       if v > 0 || w == 0 {
>           return
>       }
>   
>       //此时，counter一定等于0，而waiter一定大于0（内部维护waiter，不会出现小于0的情况），
>       //先把counter置为0，再释放waiter个数的信号量
>       *statep = 0
>       for ; w != 0; w-- {
>           runtime_Semrelease(semap, false) //释放信号量，执行一次释放一个，唤醒一个等待者
>       }
>   }
>   
>   // Wait()方法也做了两件事，一是累加waiter, 二是阻塞等待信号量
>   func (wg *WaitGroup) Wait() {
>       statep, semap := wg.state() //获取state和semaphore地址指针
>       for {
>           state := atomic.LoadUint64(statep) //获取state值
>           v := int32(state >> 32)            //获取counter值
>           w := uint32(state)                 //获取waiter值
>           if v == 0 {                        //如果counter值为0，说明所有goroutine都退出了，不需要待待，直接返回
>               return
>           }
>   
>           // 使用CAS（比较交换算法）累加waiter，累加可能会失败，失败后通过for loop下次重试
>           if atomic.CompareAndSwapUint64(statep, state, state+1) {
>               runtime_Semacquire(semap) //累加成功后，等待信号量唤醒自己
>               return
>           }
>       }
>   }
>   
>   // Done
>   func (wg *WaitGroup) Done() {
>       wg.Add(-1)
>   }
>   
>   ```
>
>   

###### Context

>   ```go
>   
>   type Context interface {
>       Deadline() (deadline time.Time, ok bool)  // 该方法返回一个deadline和标识是否已设置deadline的bool值，如果没有设置deadline，则ok == false，此时deadline为一个初始值的time.Time值
>   
>       Done() <-chan struct{} // 该方法返回一个channel，需要在select-case语句中使用，如”case <-context.Done():”。当context关闭后，Done()返回一个被关闭的管道，关闭的管道仍然是可读的，据此goroutine可以收到关闭请求；当context还未关闭时，Done()返回nil。
>   
>       Err() error
>   
>       Value(key interface{}) interface{}
>   }
>   
>   
>   // 空Context
>   type emptyCtx int
>   
>   func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
>       return
>   }
>   
>   func (*emptyCtx) Done() <-chan struct{} {
>       return nil
>   }
>   
>   func (*emptyCtx) Err() error {
>       return nil
>   }
>   
>   func (*emptyCtx) Value(key interface{}) interface{} {
>       return nil
>   }
>   
>   
>   // concelCtx
>   type cancelCtx struct {
>       Context
>   
>       mu       sync.Mutex            // protects following fields
>       done     chan struct{}         // created lazily, closed by first cancel call
>       children map[canceler]struct{} // set to nil by the first cancel call
>       err      error                 // set to non-nil by the first cancel call
>   }
>   
>   // timer Ctx
>   type timerCtx struct {
>       cancelCtx
>       timer *time.Timer  // 定最长存活时间，比如context将在30s后结束
>   
>       deadline time.Time // deadline: 指定最后期限，比如context将2018.10.20 00:00:00之时自动结束
>   }
>   
>   // valueCtx
>   type valueCtx struct {
>       Context
>       key, val interface{}
>   }
>   
>   
>   ```
>
>   -   Context仅仅是一个接口定义，根据实现的不同，可以衍生出不同的context类型；
>   -   cancelCtx实现了Context接口，通过WithCancel()创建cancelCtx实例；
>   -   timerCtx实现了Context接口，通过WithDeadline()和WithTimeout()创建timerCtx实例；
>   -   valueCtx实现了Context接口，通过WithValue()创建valueCtx实例；
>   -   三种context实例可互为父节点，从而可以组合成不同的应用形式；
>   -   

#### 反射

###### 反射的定义？

-   反射提供了一种让程序检查自身结构的能力（反射是一种检查interface变量的底层类型和值的机制）
-   反射是困惑的源泉

###### interface

-   interface类型是一种特殊的类型，它代表方法集合。 它可以存放任何实现了其方法的值。

###### Interface 类型是如何表示的？

-   

###### 反射定律

-   interface类型存储了一个（type，value）对，反射是通过检查（type，value）对
-   `reflect.Type` 提供一组接口处理interface的类型，即（value, type）中的type
-   `reflect.Value`提供一组接口处理interface的值,即(value, type)中的value

定律

-   反射可以将interface类型变量转换成反射对象

    -   ```go
        package main
        
        import (
            "fmt"
            "reflect"
        )
        
        func main() {
            var x float64 = 3.4
            t := reflect.TypeOf(x)  //t is reflect.Type
            fmt.Println("type:", t)
        
            v := reflect.ValueOf(x) //v is reflect.Value
            fmt.Println("value:", v)
        }
        
        // 输出
        type: float64
        value: 3.4
        ```

    -   

-   反射可以将反射对象还原成interface对象

    -   ```go
        package main
        
        import (
            "fmt"
            "reflect"
        )
        
        func main() {
            var x float64 = 3.4
        
            v := reflect.ValueOf(x) //v is reflect.Value
        
            var y float64 = v.Interface().(float64)
            fmt.Println("value:", y)
        }
        ```

    -   

-   反射对象可修改，value值必须是可设置的

    -   ```go
        package main
        
        import (
            "reflect"
        )
        
        func main() {
            var x float64 = 3.4
            v := reflect.ValueOf(x)
            v.SetFloat(7.1) // Error: will panic.
        }
        
        // panic
        
        package main
        
        import (
        "reflect"
            "fmt"
        )
        
        func main() {
            var x float64 = 3.4
            v := reflect.ValueOf(&x)
            v.Elem().SetFloat(7.1)      // reflect.Value提供了Elem()方法，可以获得指针向指向的value
            fmt.Println("x :", v.Elem().Interface())
        }
        
        ```

#### Go Test

###### 单元测试

###### 性能测试

###### 示例测试

#### 定时器

###### Timer



###### Ticker



#### 性能调优

#### 包管理





1：
