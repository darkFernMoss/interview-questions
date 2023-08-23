

#### make和new

1. `make`：`make` 用于创建切片（slices）、映射（maps）和通道（channels）等引用类型的对象。这些引用类型在创建时需要进行内部数据结构的初始化，以便于后续的使用。`make` 函数返回的是被初始化后的对象本身，而不是对象的指针。

2. `new`：`new` 用于创建值类型的对象，如结构体（struct）或基本数据类型，返回的是一个指向新分配的零值对象的指针。`new` 函数分配了足够的内存来存储一个对象，并将其初始化为零值。

总结起来，`make` 用于创建引用类型的对象，并进行初始化，返回对象本身；`new` 用于创建值类型的对象，并返回指向对象的指针。根据不同的对象类型和需求，选择使用适当的函数来创建对象。

#### defer 

defer执行顺序和调用顺序相反，类似于栈**后进先出**(LIFO)。

defer在return之后执行，但在函数退出之前，defer可以修改返回值。只对有名返回值适用，也就是指明返回值的名字，否则会在函数结束前拷贝一份返回值，当defer修改原值后也不影响最终的返回结果。



2.panic后的defer不会被执行（遇到panic，如果没有捕获错误，函数会立刻终止）
3.panic没有被recover时，抛出的panic到当前goroutine最上层函数时，最上层程序直接异常终止

```go
package main

import "fmt"

func F() {
	defer func() {
		fmt.Println("b")
	}()
	panic("a")
}

func main() {
	defer func() {
		fmt.Println("c")
	}()
	//子函数抛出的panic没有recover时，上层函数时，程序直接异常终止
	F()
	fmt.Println("继续执行")
}

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/e5167fc7458a4f3a8762ccdce068d4cc.png)



4.panic有被recover时，当前goroutine最上层函数正常执行

```go
package main

import "fmt"

func F() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("捕获异常", err)
		}
		fmt.Println("b")
	}()
	panic("a")
}

func main() {
	defer func() {
		fmt.Println("c")
	}()
	//子函数抛出的panic没有recover时，上层函数时，程序直接异常终止
	F()
	fmt.Println("继续执行")
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/62dc3d8557e441b292978316c7110329.png)

#### Interface 判断是否为Nil

只有interface的判断比较特殊，当interface并未初始化类型时才能==nil。即interface 只有在类型和值都为nil时才==nil。而其他类型像指针，channel，map，slice只有值是nil就可以。







#### go垃圾回收

**Go V1.3之前的标记-清除：**
1.暂停业务逻辑，找到不可达的对象，和可达对象
2.开始标记，程序找出它所有可达的对象，并做上标记
3.标记完了之后，然后开始清除未标记的对象。
4.停止暂停，让程序继续跑。然后循环重复这个过程，直到process程序生命周期结束

标记-清除的缺点：
STW（stop the world）：让程序暂停，程序出现卡顿
标记需要扫描整个heap
清除数据会产生heap碎片

为了减少STW的时间，后来对上述的第三步和第四步进行了替换。



Go V1.5 三色标记法
1.把新创建的对象，默认的颜色都标记为“白色”

2.每次GC回收开始，然后从根节点开始遍历所有对象，把遍历到的对象从白色集合放入“灰色”集合

3.遍历灰色集合，将灰色对象引用的对象从白色集合放入到灰色集合，之后将此灰色对象放入到黑色集合

4.重复第三步，直到灰色中无任何对象

5.回收所有的白色标记的对象，也就是回收垃圾

![img](https://pic1.zhimg.com/80/v2-d695718ace0d8f5db7642d96684f6174_1440w.webp)

分析bug的根源所在，主要是因为程序在运行过程中出现了下面俩种情况

1. 一个白色对象被黑色对象引用
2. 灰色对象与它可达的白色对象之间的引用关系遭到破坏



因此在此基础上拓展出了俩种方法，强三色不变式和弱三色不变式

1. 强三色不变式：不允许黑色对象引用白色对象
2. 弱三色不变式：黑色对象可以引用白色，白色对象存在其他灰色对象对他的引用，或者他的链路上存在灰色对象

为了实现这俩种不变式的设计思想，从而引出了屏障机制，即在程序的执行过程中加一个判断机制，满足判断机制则执行回调函数。

![img](https://pic1.zhimg.com/80/v2-98d9ce04e81f29155c69c282f3d55600_1440w.webp)

屏障机制分为插入屏障和删除屏障，插入屏障实现的是强三色不变式，删除屏障则实现了弱三色不变式。**值得注意的是为了保证栈的运行效率**，屏障只对堆上的内存对象启用，栈上的内存会在GC结束后启用STW重新扫描。

插入屏障：对象被引用时触发的机制，当白色对象被黑色对象引用时，白色对象被标记为灰色（栈上对象无插入屏障）。

![img](https://pic4.zhimg.com/80/v2-ff92328ca7976a791cc45738c98d7053_1440w.webp)

缺点在于：如果对象1在栈上新创建了一个对象6，由于栈没有屏障机制，所以对象6仍为白色节点会被回收



![img](https://pic1.zhimg.com/80/v2-d24e03bf25c377813ffe3118a65a9d3c_1440w.webp)



所以栈在GC迭代结束时（没有灰色节点），会对栈执行STW，重新进行扫描清除白色节点。（STW时间为10-100ms）

删除屏障：对象被删除时触发的机制。如果灰色对象引用的白色对象被删除时，那么白色对象会被标记为灰色。



![img](https://pic1.zhimg.com/80/v2-40cdc2f0e2975b77c155634ab14d2790_1440w.webp)



缺点：这种做法回收精度较低，一个对象即使被删除仍可以活过这一轮再下一轮被回收。（如果对象4没有引用对象3，此时对象3应该作为垃圾被回收，但是对象3却要等到下一轮GC才会被回收）



![img](https://pic2.zhimg.com/80/v2-ed7c3af3bde2d7e867c47c00d3e66ec1_1440w.webp)



同样也存在对栈的二次扫描影响程序的效率。

**Go1.8 三色标记+混合写屏障**

基于插入写屏障和删除写屏障在结束时需要STW来重新扫描栈，所带来的性能瓶颈，Go在1.8引入了混合写屏障的方式实现了弱三色不变式的设计方式，混合写屏障分下面四步

1. GC开始时将栈上可达对象全部标记为黑色（不需要二次扫描，无需STW）
2. GC期间，任何栈上创建的新对象均为黑色
3. 被删除引用的对象标记为灰色
4. 被添加引用的对象标记为灰色

下面为混合写屏障过程

![img](https://pic3.zhimg.com/80/v2-750e44773b0530fe7446171f90581de2_1440w.webp)

触发GC有俩个条件，一是堆内存的分配达到控制器计算的触发堆大小，初始大小环境变量GOGC，之后堆内存达到上一次垃圾收集的 2 倍时才会触发GC。二是如果一定时间内没有触发，就会触发新的循环，该触发条件由`runtime.forcegcperiod`变量控制，默认为 2 分钟。

#### GC调优

1.控制内存分配的速度，限制Goroutine的数量，提高赋值器mutator的CPU利用率（降低GC的CPU利用率）
2.少量使用+连接string
3.slice提前分配足够的内存来降低扩容带来的拷贝
4.避免map key对象过多，导致扫描时间增加
5.变量复用，减少对象分配，例如使用sync.Pool来复用需要频繁创建临时对象、使用全局变量等
6.增大GOGC的值，降低GC的运行频率



#### go内存管理

TCMalloc是Thread Cache Malloc的简称，是Go内存管理的起源，Go的内存管理是借鉴了TCMalloc

TCMalloc的做法是什么呢？为每个线程预分配一块缓存，线程申请小内存时，可以从缓存分配内存。

这样有2个好处：

1. 为线程预分配缓存需要进行1次系统调用，后续线程申请小内存时直接从缓存分配，都是在用户态执行的，没有了系统调用，缩短了内存总体的分配和释放时间，这是快速分配内存的第二个层次。
2. 多个线程同时申请小内存时，从各自的缓存分配，访问的是不同的地址空间，从而无需加锁，把内存并发访问的粒度进一步降低了，这是快速分配内存的第三个层次。



##### TCMalloc基本原理

下面就简单介绍下TCMalloc，细致程度够我们理解Go的内存管理即可。

![img](https://pic3.zhimg.com/80/v2-17a205c8fdfe0d21ab06f469df72b2fe_1440w.webp)

结合上图，介绍TCMalloc的几个重要概念：

- **Page**

操作系统对内存管理以页为单位，TCMalloc也是这样，只不过TCMalloc里的Page大小与操作系统里的大小并不一定相等，而是倍数关系。《TCMalloc解密》里称x64下Page大小是8KB。

- **Span**

一组连续的Page被称为Span，比如可以有2个页大小的Span，也可以有16页大小的Span，Span比Page高一个层级，是为了方便管理一定大小的内存区域，Span是TCMalloc中内存管理的基本单位。

- **ThreadCache**

ThreadCache是每个线程各自的Cache，一个Cache包含多个空闲内存块链表，每个链表连接的都是内存块，同一个链表上内存块的大小是相同的，也可以说按内存块大小，给内存块分了个类，这样可以根据申请的内存大小，快速从合适的链表选择空闲内存块。由于每个线程有自己的ThreadCache，所以ThreadCache访问是无锁的。

- **CentralCache**

CentralCache是所有线程共享的缓存，也是保存的空闲内存块链表，链表的数量与ThreadCache中链表数量相同，当ThreadCache的内存块不足时，可以从CentralCache获取内存块；当ThreadCache内存块过多时，可以放回CentralCache。由于CentralCache是共享的，所以它的访问是要加锁的。

- **PageHeap**

PageHeap是对堆内存的抽象，PageHeap存的也是若干链表，链表保存的是Span。当CentralCache的内存不足时，会从PageHeap获取空闲的内存Span，然后把1个Span拆成若干内存块，添加到对应大小的链表中并分配内存；当CentralCache的内存过多时，会把空闲的内存块放回PageHeap中。

如下图所示，分别是1页Page的Span链表，2页Page的Span链表等，最后是large span set，这个是用来保存中大对象的。毫无疑问，PageHeap也是要加锁的。

![img](https://pic2.zhimg.com/80/v2-8987e2cd5a39e7218b7a5c7689a1da7d_1440w.webp)

前文提到了小、中、大对象，Go内存管理中也有类似的概念，我们看一眼TCMalloc的定义：

- 小对象大小：0~256KB
- 中对象大小：257~1MB
- 大对象大小：>1MB

小对象的分配流程：ThreadCache -> CentralCache -> HeapPage，大部分时候，ThreadCache缓存都是足够的，不需要去访问CentralCache和HeapPage，无系统调用配合无锁分配，分配效率是非常高的。

中对象分配流程：直接在PageHeap中选择适当的大小即可，128 Page的Span所保存的最大内存就是1MB。

大对象分配流程：从large span set选择合适数量的页面组成span，用来存储数据。

通过本节的介绍，你应当对TCMalloc主要思想有一定了解了，我建议再回顾一下上面的内容。

![img](https://pic4.zhimg.com/80/v2-1aa731ddd77b6cad73c8f68f864ea5ef_1440w.webp)

Golang的内存管理组件主要有：mspan、mcache、mcentral和mheap

内存管理单元：mspan

mspan是内存管理的基本单元，该结构体中包含next和 prev两个字段，它们分别指向了前一个和后一个mspan，每个mspan都管理npages个大小为8KB的页，一个span是由多个page组成的，这里的页不是操作系统中的内存页，它们是操作系统内存页的整数倍。


线程缓存：mcache
mcache用Span classes作为索引管理多个用于分配的mspan ，它包含所有规格的mspan。它是_NumSizeClasses 的2倍，也就是68X2=136，其中*2是将spanClass分成了有指针和没有指针两种，方便与垃圾回收。对于每种规格，有2个mspan，一个mspan不包含指针，另一个mspan则包含指针。对于无指针对象的mspan在进行垃圾回收的时候无需进一步扫描它是否引用了其他活跃的对象。
mcache在初始化的时候是没有任何mspan资源的，在使用过程中会动态地从mcentral申请,
之后会缓存下来。当对象小于等于32KB大小时，使用mcache 的相应规格的mspan进行分配。

mcache拥有一个allocCache字段，作为一个位图，来标记span中的元素是否被分配。
当请求的mache中对应的span没有可以使用的元素时，需要从mcentral中加锁查找，分别遍历mcentral中有空闲元素的nonempty链表和没有空闲元素的empty链表。没有空闲元素的empty链表可能会有被垃圾回收标记为空闲的还未来得及清理的元素，这些元素也是可用的

中心缓存：mcentral

mcentral管理全局的mspan供所有线程使用，全局mheap变量包含central字段，每个mcentral结构都维护在mheap结构内


每个mcentral管理一种spanClass的mspan，并将有空闲空间和没有空闲空间的mspan分开管理。partial和full的数据类型为spanSet，表示mspans集，可以通过pop、push来获得mspans


简单说下mcache 从 mcentral获取和归还mspan的流程:

获取:加锁，从 partial链表找到一个可用的mspan；并将其从 partial链表删除；将取出的mspan加入到full
链表；将mspan返回给工作线程，解锁。
归还:加锁，将mspan从full链表删除；将mspan 加入到partial链表，解锁。

页堆：mheap

mheap管理Go的所有动态分配内存，可以认为是Go程序持有的整个堆空间，全局唯一



所有mcentral的集合则是存放于mheap中的。mheap里的arena区域是堆内存的抽象，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象。运行时使用二维的runtime.heapArena数组管理所有的内存，每个runtime.heapArena都会管理64MB的内存。

当申请内存时，依次经过mcache和mcentral都没有可用合适规格的大小内存，这时候会向mheap申请一块内存。然后按指定规格划分为一些列表，并将其添加到相同规格大小的 mcentral 的非空闲列表后面

分配流程：

首先通过计算使用的大小规格
然后使用mcache 中对应大小规格的块分配。
如果mcentral中没有可用的块，则向mheap申请，并根据算法找到最合适的mspan 。
如果申请到的mspan超出申请大小，将会根据需求进行切分，以返回用户所需的页数。剩余的页构成一个		
新的mspan 放回mheap 的空闲列表。
如果mheap中没有可用span，则向操作系统申请一系列新的页(最小1MB)



#### 竞态，内存逃逸

竞态、内存逃逸
1.资源竞争，就是在程序中，同一块内存同时被多个 goroutine 访问。我们使用 go build、go run、go test 命令时，添加 -race 标识可以检查代码中是否存在资源竞争。

解决这个问题，我们可以给资源进行加锁，让其在同一时刻只能被一个协程来操作。

sync.Mutex
sync.RWMutex

2.一般来说，局部变量会在函数返回后被销毁，因此被返回的引用就成为了"无所指""的引用，程序会进入未知状态。

但这在Go中是安全的，Go编译器将会对每个局部变量进行逃逸分析。如果发现局部变量的作用域超出该函数，则不会将内存分配在栈上，而是分配在堆上，因为他们不在栈区，即使释放函数，其内容也不会受影响。

```go
func add(x, y int) *int {
	res := x + y
	return &res
}
```

这个例子中，函数add局部变量res发生了逃逸。res作为返回值，在main函数中继续使用，因此res指向的内存不能够分配在栈上，随着函数结束而回收，只能分配在堆上。

go run -gcflags=-m .\main.go

逃逸场景：

1. 指针逃逸，同上举的例子。
2. 变量大小不确定。
3. 动态类型逃逸，即编译期间不确定类型的变量，如形参中传入的interface类型。

#### golang内存对齐机制

为了能让CPU可以更快的存取到各个字段，Go编译器会帮你把struct结构体做数据的对齐。所谓的数据对齐，是指内存地址是所存储数据大小（按字节为单位）的整数倍，以便CPU可以一次将该数据从内存中读取出来。编译器通过在结构体的各个字段之间填充一些空白已达到对齐的目的。

不同硬件平台占用的大小和对齐值都可能是不一样的，每个特定平台上的编译器都有自己的默认“对齐系数”，32位系统对齐系数是4，64位系统对齐系数是8
不同类型的对齐系数也可能不一样，使用Go 语言中的unsafe.Alignof函数可以返回相应类型的对齐系数，对齐系数都符合2^n这个规律，最大也不会超过8

对齐原则：

1.结构体变量中成员的偏移量必须是成员大小的整数倍
2.整个结构体的地址必须是最大字节的整数倍（结构体的内存占用是1/4/8/16 byte...）

type T2 struct{
	i8 int8
	i64 int64
	i32 int32
}

type T3 struct{
	i8 int8
	i32 int32
	i64 int64
}

![在这里插入图片描述](https://img-blog.csdnimg.cn/31b7ef13828a409283e095e056d9f12e.png)

#### golang的map的结构

hmap 结构相当于 go map 的头, 它存储了哈希桶的内存地址, 哈希桶之间在内存中紧密连续存储, 彼此之间没有额外的 gap, 每个哈希桶最多存放 8 个 k/v 对, 冲突次数超过 8 时会存放到溢出桶中, 哈希桶可以跟随多个溢出桶, 呈现一种链式结构, 当 HashTable 的装载因子超过阈值(6.5) 后会触发哈希的扩容, 避免效率下降

![img](https://pic3.zhimg.com/80/v2-e0b9187cfc0dd066239b44df1f4594ba_1440w.webp)

#### Go map 的查找

当要根据 key 从 map 中查询对应的 elem 时, 在 go 有两种写法, 一种是直接取值, 例如我们定义 hash := make(map[int]int), 则可以使用 s := hash[key], 也可以使用 s, ok := hash[key], 第一种写法无论 key 是否存在于 map 中, s 都会获取一个返回值, 而当 key 不存在时会返回对应类型的零值, 而第二种写法中, ok 变量可以标识此次是否从 map 中真正的获取到了 key 所对应的 elem, 在 go 语言底层, 这两种写法实际会调用两个不同函数, 它们都位于 src/runtime/map.go 中, 分别调用 mapaccess1 和 mapaccess2 函数, 这两个函数的内部逻辑几乎是一样的, 第二个相比于第一个仅仅多了一个是否查询到的标志位, 我们只来分析 mapaccess1 函数即可, go map 中使用了 aes 和 memhash 两类哈希, 当运行架构支持 aes 哈希时会优先使用 aes 作为 HashFunc, 具体的判定逻辑在 src/runtime/alg.go 的 alginit() 函数中, 当要 map 中查询一个元素时, go 首先使用 key 和哈希表的 hash0, 即创建 map 时生成的随机数做哈希函数运算得到哈希值, hmap 中的 B 表征了当前哈希表中的哈希桶数量, 哈希桶数量等于 2B2B, 这里 go 使用了我们在第一节提到的除留余数法来计算得到相应的哈希桶, 因为桶的数量是 2 的整数次幂, 因此这里的取余运算可以使用位运算来替代, 将哈希值与桶长度减一做按位与即得到了对应的桶编号, 当前这里的桶编号是一个逻辑编号, hmap 结构中存储了哈希桶的内存地址, 在这个地址的基础上偏移桶编号*桶长度便得到了对应的哈希桶的地址, 接下来进一步在该哈希桶中找寻 key 对应的元素, 比较的时候基于哈希值的高 8 位与桶中的 topbits 依次比较, 若相等便可以根据 topbits 所在的相对位置计算出 key 所在的相对位置, 进一步比较 key 是否相等, 若 key 相等则此次查找过程结束, 返回对应位置上 elem, 若 key 不相等, 则继续往下比较 topbits, 若当前桶中的所有 topbits 均与此次要找到的元素的 key 的哈希值的高 8 位不相等, 则继续沿着 overflow 向后探查溢出桶, 重复刚刚的过程, 直到找到对应的 elem, 或遍历完所有的溢出桶仍未找到目标元素, 此时返回该类型的零值



#### Golang的map为什么是无序的？

使用range多次遍历map时输出的key和vabue 的顺序可能不同。这是Go语言的设计者们有意为之，旨在提示开发者们，Go底层实现并不保证map遍历顺序稳定，请大家不要依赖range遍历结果顺序

主要原因有2点:

1.map在遍历时，并不是从固定的0号bucket开始遍历的，每次遍历，都会从一个随机值序号的bucket，
再从其中随机的cell开始遍历
2.map遍历时，是按序遍历bucket，同时按需遍历bucket中和其overflow bucket中的cell。但是map在扩
容后，会发生key的搬迁，这造成原来落在一个buket中的Key,搬迁后，有可能会落到其他bucket中了，
从这个角度看，遍历map的结果就不可能是按照原来的顺序了



#### 为什么Golang的Map的负载因子是6.5？

什么是负载因子?

负载因子(load factor)，用于衡量当前哈希表中空间占用率的核心指标，也就是每个bucket桶存储的平均	
元素个数。
负载因子=哈希表存储的元素个数/桶个数

另外负载因子与扩容、迁移等重新散列(rehash)行为有直接关系:

在程序运行时,会不断地进行插入、删除等，会导致 bucket不均，内存利用率低，需要迁移。
在程序运行时，出现负载因子过大，需要做扩容，解决 bucket过大的问题。

负载因子是哈希表中的一个重要指标，在各种版本的哈希表实现中都有类似的东西，主要目的是为了平衡buckets 的存储空间大小和查找元素时的性能高低，在接触各种哈希表时都可以关注一下，做不同的对比，看看各家的考量。

Go官方发现:装载因子越大，填入的元素越多，空间利用率就越高，但发生哈希冲突的几率就变大。反之，装载因子越小，填入的元素越少，冲突发生的几率减小，但空间浪费也会变得更多，而且还会提高扩容操作的次数
根据这份测试结果和讨论，Go官方取了一个相对适中的值，把Go中的 map的负载因子硬编码为6.5，这就是6.5的选择缘由。
这意味着在Go语言中，当map存储的元素个数大于或等于6.5*桶个数时，就会触发扩容行为。



#### Golang map如何扩容

在向map插入新key的时候，会进行条件检测，符合下面这2个条件，就会触发扩容

```go
if !h.growing() &&(overLoadFactor(h.count+1,h.B)||tooManyOverflowBuckets(h.noverflow， h.8)){
hashGrow(t, h)
goto again // Growing the table invalidates everything, so try again
}

//判断是否在扩容
func (h *hmap) growing( ) bool {
return h.oldbuckets != nil
}
```



扩容条件：

```go
条件1：超过负载 map元素个数>6.5*桶个数
func overLoadFactor(count int, B uint8) bool{
	return count > bucketCnt && uintptr(count)>loadFactor*bucketShift(B)
}
其中

bucketCnt=8，一个桶可以装的最大元素个数
loadFactor=6.5，负载因子，平均每个桶的元素个数
bucketShift(8), 桶的个数

条件2：溢出桶太多
当桶总数<2^15时，如果溢出桶总数>=桶总数，则认为溢出桶过多
当桶总数>=2^15时，直接与2^15比较，当溢出桶总数>=2^15时，即认为溢出桶太多了
```

#### Channel的实现原理

概念:

Go中的channel是一个队列，遵循先进先出的原则，负责协程之间的通信(Go语言提倡不要通过共享内存
来通信，而要通过通信来实现内存共享，CSP(CommunicatingSequential Process)并发模型，就是通过
goroutine和channel来实现的)

前面提及过 channel 创建后返回了 **hchan** 结构体，现在我们来研究下这个结构体，它的主要字段如下：

```text
type hchan struct {
 qcount   uint   // channel 里的元素计数
 dataqsiz uint   // 可以缓冲的数量，如 ch := make(chan int, 10)。 此处的 10 即 dataqsiz
 elemsize uint16 // 要发送或接收的数据类型大小
 buf      unsafe.Pointer // 当 channel 设置了缓冲数量时，该 buf 指向一个存储缓冲数据的区域，该区域是一个循环队列的数据结构
 closed   uint32 // 关闭状态
 sendx    uint  // 当 channel 设置了缓冲数量时，数据区域即循环队列此时已发送数据的索引位置
 recvx    uint  // 当 channel 设置了缓冲数量时，数据区域即循环队列此时已接收数据的索引位置
 recvq    waitq // 想读取数据但又被阻塞住的 goroutine 队列
 sendq    waitq // 想发送数据但又被阻塞住的 goroutine 队列

 lock mutex
 ...
}
```

channel 在进行读写数据时，会根据无缓冲、有缓冲设置进行对应的阻塞唤起动作，它们之间还是有区别的。下面我们来捋一下这些不同之处。

等待队列：
双向链表，包含一个头节点和一个尾节点
每个节点是一个sudog结构体变量，记录哪个协程在等待，等待的是哪个channel，等待发送/接收的数据在哪里

```go
type waitq struct{
	first *sudog
	last *sudog
}

type sudog struct{
	g *g //哪个协程在等待
	next *sudog
	prev *sudog
	elem unsafe.Pointer //等待发送/接收的数据在哪里
	c *hchan //等待的是哪个channel
	...
}
```

![img](https://img2018.cnblogs.com/blog/614799/201907/614799-20190718102346011-391234902.png)

##### **无缓冲 channel**

由于对 channel 的读写先后顺序不同，处理也会有所不同，所以，还得再进一步区分：

##### **channel 先写再读**

在这里，我们暂时认为有 2 个 goroutine 在使用 channel 通信，按先写再读的顺序，则具体流程如下：

![img](https://pic2.zhimg.com/80/v2-f2738822fbf052dff0ee47cb225fc405_1440w.webp)

可以看到，由于 channel 是无缓冲的，所以 G1 暂时被挂在 sendq 队列里，然后 G1 调用了 gopark 休眠了起来。

接着，又有 goroutine 来 channel 读取数据了：

![img](https://pic3.zhimg.com/80/v2-451480d87d1d19fa72fce11779ef7742_1440w.webp)

此时 G2 发现 sendq 等待队列里有 goroutine 存在，于是直接从 G1 copy 数据过来，并且会对 G1 设置 goready 函数，这样下次调度发生时， G1 就可以继续运行，并且会从等待队列里移除掉。

##### **channel 先读再写**

先读再写的流程跟上面一样。

![img](https://pic3.zhimg.com/80/v2-b18555f9ae91578c29213fade271e97a_1440w.webp)

G1 暂时被挂在了 recvq 队列，然后休眠起来。

G2 在写数据时，发现 recvq 队列有 goroutine 存在，于是直接将数据发送给 G1。同时设置 G1 goready 函数，等待下次调度运行。

![img](https://pic4.zhimg.com/80/v2-d3692a423f1a347213ec28d0bb3e921b_1440w.webp)

##### **有缓冲 channel**

在分析完了无缓冲 channel 的读写后，我们继续看看**有缓冲 channel 的读写**。同样的，我们分为 2 种情况：

##### **channel 先写再读**

这一次会优先判断缓冲数据区域是否已满，如果未满，则将数据保存在缓冲数据区域，即环形队列里。如果已满，则和之前的流程是一样的。



![img](https://pic4.zhimg.com/80/v2-53c29200c3be1f606ba80741d7979a0f_1440w.webp)

当 G2 要读取数据时，会优先从缓冲数据区域去读取，并且在读取完后，会检查 sendq 队列，如果 goroutine 有等待队列，则会将它上面的 data 补充到缓冲数据区域，并且也对其设置 goready 函数。

![img](https://pic2.zhimg.com/80/v2-065bd4af22ea59db02c1ed61686e5e89_1440w.webp)

##### **channel 先读再写**

此种情况和无缓冲的先读再写是一样流程，此处不再重复说明。



#### channel是并发安全的：

```go
通道的发送和接收操作是原子的，即一个完整的发送或接收操作是一个原子操作，不会被其他goroutine中断。

当一个goroutine向channel发送数据时，如果channel已满，则发送操作会被阻塞，直到有其他goroutine从该channel中接收数据后释放
空间，发送操作才能继续执行。在这种情况下，channel内部会获取一个锁，保证只有一个goroutine能够往其中写入数据。

同样地，当一个goroutine从channel中接收数据时，如果channel为空，则接收操作会被阻塞，直到有其他goroutine向该channel中发送
数据后才能继续执行。在这种情况下，channel内部也会获取一个锁，保证只有一个goroutine能够从其中读取数据。

```

#### Channel死锁场景

1.非缓存channel只写不读
2.非缓存channel读在写后面
3.缓存channel写入超过缓冲区数量
4.空读
5.多个协程相互等待



#### Golang互斥锁的实现原理

互斥锁对应的底层结构是sync.Mutex结构体，位于src/sync/mutex.go中

```
type Mutex struct{
	state int32
	sema uint32
}
```

state表示锁的状态，有锁定、被唤醒、饥饿模式等，并且是用state的二进制来标识的，不同模式下会有不同的处理方式

state字段表示当前互斥锁的状态信息，它是int32类型，其低三位的二进制位均有相应的状态含义。

> mutexLocked是state中的低1位，用二进制表示为0001(为了方便，这里只描述后4位)，它代表该互斥锁
> 是否被加锁。
> mutexwoken是低2位，用二进制表示为0010，它代表互斥锁上是否有被唤醒的	
> goroutine
> mutexstarving是低3位，用二进制表示为0100，它代表当前互斥锁是否处于饥饿模式。
> state剩下的29位用于统计在互斥锁上的等待队列中goroutine数目(waiter)。
>
> sema表示信号量，mutex阻塞队列的定位是通过这个变量来实现的，从而实现goroutine的阻塞和唤醒
> sema 锁是信号量/信号锁。
> 核心是一个 uint32 值，含义是同时可并发的数量。
> 每一个 sema 锁都对应一个 SemaRoot 结构体，其中有一个平衡二叉树用于协程队列：
> 获取 sema 锁，其实就是将一个 uint32 的值减一，如果这个操作成功，便获取到锁。
> 释放 sema 锁，将 unit32 加一 atomic.Xadd(addr, 1)，如果这个操作成功，便获释放锁。
> 获取锁的时候，如果 uint32 值一开始就为 0，或减到了 0，则协程休眠: goparkunlock()，进入堆树等待：

##### 加锁过程：

1.通过原子操作CAS获取锁
2.获取失败则判断是否满足自旋条件，满足则进入自旋状态
3.自旋超过一定次数之后获取sema，获取sema成功后将其记录在平衡树上，并将WaiterShift值置为1，进
入阻塞状态，等待被唤醒
4.锁被释放后会唤醒进入阻塞态的goroutine，被唤醒的goroutine重试之前的操作
5.请求锁的等待时间过长，则进入饥饿模式，不用抢直接拿到锁


##### 解锁过程：

1.通过原子操作add解锁
2.如果仍有goroutine在等待，唤醒等待的goroutine
3.如果是饥饿模式下，让其直接获取到锁

Goroutine的枪锁模式

##### 1.正常模式(非公平锁)

1.在刚开始的时候，是处于正常模式(Barging)，也就是，当一个G1持有着一个锁的时候，G2会自旋地去	
尝试获取这个锁
2.当自旋超过4次还没有能获取到锁的时候，这个G2就会被加入到获取锁的等待队列里面，并阻塞等待
唤醒goroutine 竞争锁，新请求锁的 goroutine具有优势：它正在CPU上执行，而且可能有好几个，所以刚
唤醒的 goroutine有很大可能在锁竞争中失败，长时间获取不到锁，就会切换  到饥饿模式


##### 2.饥饿模式(公平锁)

当一个goroutine等待锁时间超过1毫秒时，它可能会遇到饥饿问题。在版本1.9中，这种场景下Go Mutex切换到饥饿模式(handoff)，解决饥饿问题。

starving = runtime_nanotime( )-waitStartTime > 1e6

队列中排在第一位的goroutine(队头)，同时饥饿模式下，新进来的goroutine不会参与抢锁也不会进入自旋
状态，会直接进入等待队列的尾部,这样很好的解决了老的goroutine一直抢不到锁的场景

那么也不可能说永远的保持一个饥饿的状态，总归会有吃饱的时候，也就是总有那么一刻Mutex会回归到正常模式，那么回归正常模式必须具备的条件有以下几种:

1.G的等待的时间是否小于1ms
2.等待队列已经全部清空了

当满足上述两个条件的任意一个的时候，Mutex会切换回正常模式，而Go的抢锁的过程，就是在这个正常模式和饥饿模式中来回切换进行的.

delta := int32(mutexLocked - 1<<mutexwaiterShift)
if !starving || old>>mutexwaiterShift == 1 {
delta -= mutexStarving
}
atomic.AddInt32(&m.state, delta)





#### Goroutine的实现原理

Goroutine可以理解为一种Go语言的协程（轻量级线程），是Go支持高并发的基础，属于用户态的线程，由用户管理而不是操作系统。

底层数据结构：

```go
type g struct {
 goid int64 //唯一的goroutine的ID
 sched gobuf //goroutine切换时，用于保存g的上下文
 stack stack //栈
gopc //pc of go statement that created this goroutine
 startpc uintptr  //pc of goroutine function
 ...
}
type gobuf struct {
 sp uintptr //栈指针位置
 pc uintptr //运行到的程序位置
 g guintptr //指向goroutine
 ret uintptr //保存系统调用的返回值
 ...
}
type stack struct {
lo uintptr //栈的下界内存地址
hi uintptr //栈的上界内存地址
}

```

goroutine的状态流转：

![在这里插入图片描述](https://img-blog.csdnimg.cn/6566c8588461467cbf2836bd9b99ac7e.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6e257e183034442985cabdf0f7449588.png)

1.创建：go关键字会调用底层函数runtime.newproc()创建一个goroutine，调用该函数之后，goroutine会被设置成runnable状态

> 创建好的这个goroutine会新建一个自己的栈空间，同时在G的sched中维护栈地址与程序计数器这些信
> 息。
> 每个G在被创建之后，都会被优先放入到本地队列中，如果本地队列已经满了，就会被放入到全局队列
> 中。

#### 怎么查看Goroutine的数量？

runtime.NumGoroutine()



#### Go的Slice如何扩容？

go1.18之前：
1.首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）。
2.否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍。
3.否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的1.25倍，直到新最终容量大于等于新申请的容量
4.如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）。

go1.18之后：
低于256，每次扩容两倍。超过256，不再是每次扩容1/4，而是每次增加（旧容量+3\*256）/4即1.25倍+192

#### Go 1.18版本切片扩容

##### go1.20 slice 扩容

0 0
1 1
2 2
3 4
5 8
9 16
17 32
33 64
65 128
129 256
257 512
513 848
849 1280
1281 1792
1793 2560
2561 3408
3409 5120
5121 7168
7169 9216
9217 12288

Go1.18不再以1024为临界点，而是设定了一个值为256的`threshold`，以256为临界点；超过256，不再是每次扩容1/4，而是每次增加（旧容量+3\*256）/4；

- 当新切片需要的容量cap大于两倍扩容的容量，则直接按照新切片需要的容量扩容；
- 当原 slice 容量 < threshold 的时候，新 slice 容量变成原来的 2 倍；
- 当原 slice 容量 > threshold，进入一个循环，每次容量增加（旧容量+3*threshold）/4。

如下代码所示：

```go
go复制代码//1.18
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
  newcap = cap
} else {
  const threshold = 256
  if old.cap < threshold {
    newcap = doublecap
  } else {
    // Check 0 < newcap to detect overflow
    // and prevent an infinite loop.
    for 0 < newcap && newcap < cap {
      // Transition from growing 2x for small slices
      // to growing 1.25x for large slices. This formula
      // gives a smooth-ish transition between the two.
      newcap += (newcap + 3*threshold) / 4
    }
    // Set newcap to the requested cap when
    // the newcap calculation overflowed.
    if newcap <= 0 {
      newcap = cap
    }
  }
}
```

在1.18中，优化了切片扩容的策略，让底层数组大小的增长更加平滑： 通过减小阈值并固定增加一个常数，使得优化后的扩容的系数在阈值前后不再会出现从2到1.25的突变，该commit作者给出了几种原始容量下对应的“扩容系数”：

| oldcap | 扩容系数 |
| ------ | -------- |
| 256    | 2.0      |
| 512    | 1.63     |
| 1024   | 1.44     |
| 2048   | 1.35     |
| 4096   | 1.30     |

可以看到，Go1.18的扩容策略中，随着容量的增大，其扩容系数是越来越小的，可以更好地节省内存。

我们可以试着求一个极限，当oldcap远大于256的时候，扩容系数将会变成1.25。





#### interface底层原理

```go
type iface struct {
tab *itab	//存储了接口的类型
data unsafe.Pointer //存储了接口中动态类型的数据指针
}
```

```go
type itab struct {
inter *interfacetype //类型的静态类型信息，比如io.Reader
_type *_type  //接口存储的动态类型
hash uint32 //接口动态类型的唯一标识，_type中hash的副本
_ [4]byte  //用于内存对齐
fun [1]uintptr //动态类型的函数指针列表

```

```go
type _type struct {
    size       uintptr  // 类型大小
    ptrdata    uintptr	//偏移量
    hash       uint32   // 哈希
    tflag      tflag 	//标志
    align      uint8    // 对齐方式
    fieldAlign uint8
    kind       uint8    // 类别
    equal      func(unsafe.Pointer, unsafe.Pointer) bool
    gcdata     *byte
    str        nameOff
    ptrToThis  typeOff
}
```

```go
type interfacetype struct{
	typ _type	
	pkgpath name //接口所在的包名
	mhdr []imethod  //接口中暴漏的方法在最终可执行文件中的名字和类型的偏移量
}
```

从iface或itab都可以看出，接口interface包含有两种类型：

> 一种是接口自身的类型，称为接口的静态类型，比如io.Reader等，用于确定接口类型，直接存储在itab结
> 构体中
> 一种是接口所持有数据的类型，称为接口的动态类型，用于在反射或者接口转换时确认所持有数据的实际
> 类型，存储在运行时runtime/_type结构体中。

hash是一个uint32类型的值，实际上可以看作是类型的校验码，当将接口转换成具体的类型的时候，会通过比较二者的hash值确定是否相等，只有hash相等 才能进行转换。

fun最终指向的是接口的方法集，即存储了接口所有方法的函数的指针。通过比较接口的方法集和类型的方法集，可以用来判断该类型是否实现了该接口。 把fun指向的方法集看作是一个虚函数表，也是很贴切的。

#### 空接口：

```go
type eface struct{ // 两个指针，16byte
    _type *_type             // 指向一个内部表
    data unsafe.Pointer   // 指向所持有的数据
}
```

1.空接口是单独的唯一的一种接口类型，因此自然不需要itab中的接口类型字段了
2.空接口也没有任何的方法，因此自然也不存在itab中的方法集了

#### go 打印时 %v %+v %#v 的区别？

%v 只输出所有的值；
%+v 先输出字段名字，再输出该字段的值；
%#v 先输出结构体名字值，再输出结构体（字段名字+字段的值）；

```go
package main
import "fmt"

type student struct {
 id   int32
 name string
}

func main() {
 a := &student{id: 1, name: "微客鸟窝"}

 fmt.Printf("a=%v \n", a) // a=&{1 微客鸟窝} 
 fmt.Printf("a=%+v \n", a) // a=&{id:1 name:微客鸟窝} 
 fmt.Printf("a=%#v \n", a) // a=&main.student{id:1, name:"微客鸟窝"}
}
```

#### 什么是 rune 类型？

Go语言的字符有以下两种：

1.uint8 类型，或者叫 byte 型，代表了 ASCII 码的一个字符。
2.rune 类型，代表一个 UTF-8 字符，当需要处理中文、日文或者其他复合字符时，则需要用到 rune 类型。rune 类型等价于 int32 类型。

#### golang值接收者和指针接收者的区别

无论接收器的类型，传入时如果类型不一样都会自动取地址或取值。

如果方法的接收者是指针类型，无论调用者是对象还是对象指针，修改的都是对象本身，会影响调用者

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者

通常我们使用指针类型作为方法的接收者的理由：

> 1.使用指针类型能够修改调用者的值
> 2.使用指针类型可以避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做更加高效

#### 引用传递和值传递

什么是引用传递?
将实参的地址传递给形参，函数内对形参值内容的修改，将会影响实参的值内容。Go语言是没有引用传递的，在C++中，函数参数的传递方式有引用传递。
Go的值类型(int、struct等）、引用类型（指针、slice、map、 channel)

```go
func add(x *int) {
	fmt.Printf("函数里指针地址：%p\n", &x)
	fmt.Printf("函数里指针指向变量的地址：%p\n", x)
	*x += 2
}

func main() {
	a := 2
	p := &a
	fmt.Printf("原始指针地址：%p\n", &p)
	fmt.Printf("原始指针指向变量的地址：%p\n", p)
	add(p)
	fmt.Println(a)
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/1aa1c5ed0d2142e39dbb140616bf5998.png)

这说明函数调用时，传递的依然是指针的副本，依然是值传递，但指针指向的是同一个地址，所以能修改值





#### Golang字符串拼接对比

+号拼接：因为golang的字符串是静态的，所以每次+都会重新分配一个内存空间存相加的两个字符串
fmt.Sprintf：主要是使用到了反射
Strings.Builder：

type Builder struct {
    addr *Builder // of receiver, to detect copies by value
    buf  []byte // 1
}

addr字段主要是做copycheck，buf字段是一个byte类型的切片，这个就是用来存放字符串内容的，提供的writeString()方法就是像切片buf中追加数据。

func (b *Builder) WriteString(s string) (int, error) {
 b.copyCheck()
 b.buf = append(b.buf, s...)
 return len(s), nil
}

[]byte的申请是成倍的，例如，初始大小为 0，当第一次写入大小为 10 byte 的字符串时，则会申请大小为 16 byte 的内存（恰好大于 10 byte 的 2 的指数），第二次写入 10 byte 时，内存不够，则申请 32 byte 的内存，第三次写入内存足够，则不申请新的，以此类推。
提供的String方法就是将[]byte转换为string类型，这里为了避免内存拷贝的问题，使用了强制转换来避免内存拷贝。

func (b *Builder) String() string {
 return *(*string)(unsafe.Pointer(&b.buf))
}

bytes.Buffer：strings.Builder 和 bytes.Buffer 底层都是 []byte 数组，但 strings.Builder 性能比 bytes.Buffer 略快约 10% 。一个比较重要的区别在于，bytes.Buffer 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 strings.Builder 直接将底层的 []byte 转换成了字符串类型返回了回来。

#### String和[]byte的区别

string类型本质也是一个结构体，定义如下：

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

string类型底层是一个byte类型的数组，stringStruct和slice还是很相似的，str指针指向的是byte数组的首地址，len代表的就是数组长度。

string和byte的区别：

string类型为什么还要在数组的基础上再进行一次封装呢？
这是因为在Go语言中string类型被设计为不可变的，不仅是在Go语言，其他语言中string类型也是被设计为不可变的，这样的好处就是：在并发场景下，我们可以在不加锁的控制下，多次使用同一字符串，在保证高效共享的情况下而不用担心安全问题。






