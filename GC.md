# GC

### 目录
1. Introduction
1. Mark & Sweep
2. Tri-color Mark & Sweep
3. Write Barrier
4. Stop The World

## 引言
现代高级编程语言管理内存的方式分为两种：自动和手动，像 C、C++ 等编程语言使用手动管理内存的方式，工程师编写代码过程中需要主动申请或者释放内存；而 PHP、Java 和 Go 等语言使用自动的内存管理系统，有内存分配器和垃圾收集器来代为分配和回收内存，其中垃圾收集器就是我们常说的 GC。主流的垃圾回收算法： 引用计数 追踪式垃圾回收.

<img src="https://user-images.githubusercontent.com/35750880/112945835-5f368680-9167-11eb-8fa7-941bdf0e35ec.png" height = "300" align=center />

Go 现在用的三色标记法就属于追踪式垃圾回收算法的一种。

## Mark & Sweep
### 概念
* STW:
stop the world, GC 的一些阶段需要停止所有的 mutator 以确定当前的引用关系。这便是很多人对 GC 担心的来源，这也是 GC 算法优化的重点。
* Root:
根对象是 mutator 不需要通过其他对象就可以直接访问到的对象。比如全局对象，栈对象中的数据等。通过Root对象。可以追踪到其他存活的对象。

<img src="https://user-images.githubusercontent.com/35750880/112945902-76757400-9167-11eb-8b59-e284fde904b3.png" height = "300" alt="图片名称" align=center />

### Mark & Sweep
严格按照追踪式算法的思路来实现：
1. Stop the World
2. Mark：通过 Root 和 Root 直接间接访问到的对象， 来寻找所有可达的对象，并进行标记。
3. Sweep：对堆对象迭代，已标记的对象置位标记。所有未标记的对象加入freelist， 可用于再分配。
4. Start the World

如果存在一条从根出发的指针链最终可指向某个对象，就认为这个对象是存活的。这样，未能证明存活的对象就可以标记为死亡了.该算法最大的问题是 GC 执行期间需要把整个程序完全暂停，朴素的 Mark Sweep 是整体 STW，并且分配速度慢，内存碎片率高。

<img src="https://user-images.githubusercontent.com/35750880/112945928-82f9cc80-9167-11eb-9731-ea28dc9b893c.png" height = "300" alt="图片名称" align=center />

在Go v1.1中，STW 可能秒级

标记过程需的要 STW，因为对象引用关系如果在标记阶段做了修改，会影响标记结果的正确性。 并发 GC 分为两层含义： 
* 每个 mark 或 sweep 本身是多个线程(协程)执行的(concurrent) 
* mutator 和 collector 同时运行(background) 

concurrent 这一层是比较好实现的, GC 时整体进行STW，那么对象引用关系不会再改变，对 mark 或者sweep 任务进行分块，就能多个线程(协程) conncurrent 执行任务 mark 或 sweep。

<img src="https://user-images.githubusercontent.com/35750880/112946057-ad4b8a00-9167-11eb-9975-7f8c0b3037b7.png" height = "300" alt="图片名称" align=center />

而对于 backgroud 这一层, 也就是说 mutator 和 mark，sweep 同时运行，则相对复杂。
* 1.3以前的版本使用标记-清扫的方式，整个过程都需要 STW。
* 1.3版本分离了标记和清扫的操作，标记过程STW，清扫过程并发执行。
backgroup sweep 是比较容易实现的，因为 mark 后，哪些对象是存活，哪些是要被 sweep 是已知的，sweep 的是不再引用的对象。sweep 结束前，这些对象不会再被分配到，所以 sweep 和 mutator 运行共存。无论全局还是栈不可能能访问的到这些对象，可以安全清理。 
* 1.5版本在标记过程中使用三色标记法。标记和清扫都并发执行的，但标记阶段的前后需要 STW 一定时间来做 GC 的准备工作和栈的re-scan。
## Tri-color Mark & Sweep
三色标记是对标记清楚法的改进，标记清楚法在整个执行时要求长时间 STW，Go 从1.5版本开始改为三色标记法，初始将所有内存标记为白色，然后将 roots 加入待扫描队列(进入队列即被视为变成灰色)，然后使用并发 goroutine 扫描队列中的指针，如果指针还引用了其他指针，那么被引用的也进入队列，被扫描的对象视为黑色。
* 白色对象：潜在的垃圾，其内存可能会被垃圾收集器回收。
* 黑色对象：活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象，垃圾回收器不会扫描这些对象的子对象。
* 灰色对象 ：活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象。

<img src="https://user-images.githubusercontent.com/35750880/112946241-de2bbf00-9167-11eb-8cdd-998cbda5e5c0.png" height = "300" alt="图片名称" align=center />

Go v1.5，并行 Mark & Sweep

垃圾收集器从 root 开始然后跟随指针递归整个内存空间。分配于 noscan 的 span 的对象, 不会进行扫描。然而，此过程不是由同一个 goroutine 完成的，每个指针都排队在工作池中 然后，先看到的被标记为工作协程的后台协程从该池中出队，扫描对象，然后将在其中找到的指针排入队列。

<img src="https://user-images.githubusercontent.com/35750880/112946287-ea178100-9167-11eb-85c5-6693bd1e60a7.png" height = "300" alt="图片名称" align=center />

### 染色流程
* 一开始所有对象被认为是白色
* 根节点(stacks，heap，global variables)被染色为灰色

一旦主流程走完，gc会：

* 选一个灰色对象，标记为黑色
* 遍历这个对象的所有指针，标记所有其引用的对象为灰色

最终直到所有对象需要被染色。

<img src="https://user-images.githubusercontent.com/35750880/112946346-fd2a5100-9167-11eb-8eca-90645eeb3478.png" height = "300" alt="图片名称" align=center />
<img src="https://user-images.githubusercontent.com/35750880/112946365-01ef0500-9168-11eb-823e-afc8022e8b55.png" height = "300" alt="图片名称" align=center />

如果对象位于标为noscan，则可以将其涂成黑色，并不需要对其进行扫描。

<img src="https://user-images.githubusercontent.com/35750880/112946400-0ddac700-9168-11eb-9472-f48e08a6bd35.png" height = "300" alt="图片名称" align=center />
<img src="https://user-images.githubusercontent.com/35750880/112946419-116e4e00-9168-11eb-817d-997608df7745.png" height = "300" alt="图片名称" align=center />

## Write Barrier
### 背景
1.5版本在标记过程中使用三色标记法。回收过程主要有四个阶段，其中，标记和清扫都并发执行的，但标记阶段的前后需要 STW 一定时间来做GC 的准备工作和栈的 re-scan。
* 强三色不变性：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象。
* 弱三色不变性 ：黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径。

<img src="https://user-images.githubusercontent.com/35750880/112946486-24811e00-9168-11eb-9653-17ce994e48a0.png)" height = "300" alt="图片名称" align=center />

一个白色对象被黑色对象引用，是注定无法通过这个黑色对象来保证自身存活的，与此同时，如果所有能到达它的灰色对象与它之间的可达关系全部遭到破坏，那么这个白色对象必然会被视为垃圾清除掉。 故当上述两个条件同时满足时，就会出现对象丢失的问题。如果这个白色对象下游还引用了其他对象，并且这条路径是指向下游对象的唯一路径，那么他们也是必死无疑的。

<img src="https://user-images.githubusercontent.com/35750880/112946592-4c708180-9168-11eb-9e9c-9e0507b66300.png)" height = "300" alt="图片名称" align=center />

为了防止这种现象的发生，最简单的方式就是 STW，直接禁止掉其他用户程序对对象引用关系的干扰，但是 STW 的过程有明显的资源浪费，对所有的用户程序都有很大影响，如何能在保证对象不丢失的情况下合理的尽可能的提高 GC 效率，减少 STW 时间呢？

1. 内存屏障只是对应一段特殊的代码；
2. 内存屏障这段代码在编译期间生成；
3. 内存屏障本质上在运行期间拦截内存写操作，相当于一个 hook 调用。

### Write Barrier - Dijkstra 写屏障 （Go1.5）
插入屏障拦截将白色指针插入黑色对象的操作，标记其对应对象为灰色状态，这样就不存在黑色对象引用白色对象的情况了，满足强三色不变式，在插入指针 f 时将 C 对象标记为灰色。

<img src="https://user-images.githubusercontent.com/35750880/112946788-83469780-9168-11eb-8f97-72ba496ef8ab.png" height = "300" alt="图片名称" align=center />

如果对栈上的写做拦截，那么流程代码会非常复杂，并且性能下降会非常大，得不偿失。

```
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```
#### 流程
1. 初始化 GC 任务，包括开启写屏障(write barrier)和开启辅助 GC(mutator assist)，统计 root 对象的任务数量等，这个过程需要STW。
2. 扫描所有 root 对象，包括全局指针和 goroutine(G) 栈上的指针(扫描对应 G 栈时需停止该 G)，将其加入标记队列(灰色队列)，并循环处理灰色队列的对象，直到灰色队列为空，该过程后台并行执行。
3. 完成标记工作，重新扫描(re-scan)全局指针和栈。因为 Mark 和 mutator 是并行的，所以在 Mark 过程中可能会有新的对象分配和指针赋值，这个时候就需要通过写屏障(write barrier)记录下来，re-scan 再检查一下，这个过程也是会 STW 的。
4. 按照标记结果回收所有的白色对象，该过程后台并行执行。

<img src="https://user-images.githubusercontent.com/35750880/112946808-88a3e200-9168-11eb-9351-65cd9bfd405f.png" height = "300" alt="图片名称" align=center />

### Write Barrier - Yuasa 删屏障 
删除屏障也是拦截写操作的，但是是通过保护灰色对象到白色对象的路径不会断来实现的。如上图例中，在删除指针 e 时将对象 C 标记为灰色，这样 C 下游的所有白色对象，即使会被黑色对象引用，最终也还是会被扫描标记的，满足了弱三色不变式。这种方式的回收精度低，一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮 GC 中被清理掉。

<img src="https://user-images.githubusercontent.com/35750880/112946816-8b9ed280-9168-11eb-8281-651c69374446.png" height = "300" alt="图片名称" align=center />
```
writePointer(slot, ptr):
    shade(*slot)
```
[^_^]:
    shade(*slot)是删除写屏障的变形，例如，一个堆上的灰色对象B，引用白色对象C，在GC并发运行的过程中，如果栈已扫描置黑，而赋值器将指向C的唯一指针从B中删除，并让栈上其他对象引用它，这时，写屏障会在删除指向白色对象C的指针的时候就将C对象置灰，就可以保护下来了，且它下游的所有对象都处于被保护状态。如果对象B在栈上，引用堆上的白色对象C，将其引用关系删除，且新增一个黑色对象到对象C的引用，那么就需要通过shade(ptr)来保护了，在指针插入黑色对象时会触发对对象C的置灰操作。如果栈已经被扫描过了，那么栈上引用的对象都是灰色或受灰色保护的白色对象了，所以就没有必要再进行这步操作。

### 混合屏障
插入屏障和删除屏障各有优缺点，Dijkstra 的插入写屏障在标记开始时无需 STW，可直接开始，并发进行，但结束时需要 STW 来重新扫描栈，标记栈上引用的白色对象的存活；Yuasa 的删除写屏障则需要在 GC 开始时 STW 扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象，但结束时无需 STW。

golang中的混合写屏障满足的是变形的弱三色不变式，同样允许黑色对象引用白色对象，白色对象处于灰色保护状态，但是只由堆上的灰色对象保护。

[^_^]:
    shade(*slot)是删除写屏障的变形，例如，一个堆上的灰色对象B，引用白色对象C，在GC并发运行的过程中，如果栈已扫描置黑，而赋值器将指向C的唯一指针从B中删除，并让栈上其他对象引用它，这时，写屏障会在删除指向白色对象C的指针的时候就将C对象置灰，就可以保护下来了，且它下游的所有对象都处于被保护状态。如果对象B在栈上，引用堆上的白色对象C，将其引用关系删除，且新增一个黑色对象到对象C的引用，那么就需要通过shade(ptr)来保护了，在指针插入黑色对象时会触发对对象C的置灰操作。如果栈已经被扫描过了，那么栈上引用的对象都是灰色或受灰色保护的白色对象了，所以就没有必要再进行这步操作。

```
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

由于结合了 Yuasa 的删除写屏障和 Dijkstra 的插入写屏障的优点，只需要在GC的准备阶段STW（开启写屏障和开启辅助GC，统计root 对象的任务数量等），之后发扫描各个goroutine 的栈，使其变黑并一直保持，这个过程不需要 STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行 re-scan 操作了，减少了 STW 的时间。

<img src="https://user-images.githubusercontent.com/35750880/112946876-9eb1a280-9168-11eb-88de-652be1f13a16.png)" height = "300" alt="图片名称" align=center />

为了移除栈的重扫描过程，除了引入混合写屏障之外，在垃圾收集的标记阶段，我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。

## Sweep
Sweep 让 Go 知道哪些内存可以重新分配使用，然而，Sweep 过程并不会处理释放的对象内存置为0(zeroing the memory)。而是在分配重新使用的时候，重新 reset bit。

<img src="https://user-images.githubusercontent.com/35750880/112946888-a5401a00-9168-11eb-8b0a-bd47566044d9.png" height = "300" align=center />

每个 span 内有一个 bitmap ```allocBits```，他表示上一次 GC 之后每一个 object 的分配情况，1：表示已分配，0：表示未使用或释放。
内部还使用了 ```uint64 allocCache(deBruijn)```，加速寻找 freeobject。

<img src="https://user-images.githubusercontent.com/35750880/112946896-a83b0a80-9168-11eb-8b5c-26a46ab70aa7.png" height = "300" alt="图片名称" align=center />

GC 将会启动去释放不再被使用的内存。在标记期间，GC 会用一个位图 gcmarkBits 来跟踪在使用中的内存。

<img src="https://user-images.githubusercontent.com/35750880/112946909-acffbe80-9168-11eb-9aed-aeadd37ee37a.png" height = "300" align=center />

正在被使用的内存被标记为黑色，然而当前执行并不能够到达的那些内存会保持为白色。
现在，我们可以使用 gcmarkBits 精确查看可用于分配的内存。Go 使用 gcmarkBits 赋值了 allocBits，这个操作就是内存清理。

<img src="https://user-images.githubusercontent.com/35750880/112946980-c86ac980-9168-11eb-9d6e-c05669536b17.png)" height = "300" alt="图片名称" align=center />

然而必须每个 span 都来一次类似的处理，需要耗费大量时间。Go 的目标是在清理内存时不阻碍执行，并为此提供了两种策略。
* 在后台启动一个 worker 等待清理内存，一个一个 mspan 处理
* 当申请分配内存时候 lazy 触发

清理内存段的第二种方式是即时执行。但是，由于这些内存段已经被分发到每一个处理器 P 的本地缓存 mcache 中，因此很难追踪首先清理哪些内存。这就是为什么 Go 首先将所有内存段移动到 mcentral 的原因。然后，它将会让本地缓存 mcache 再次请求它们，去即时清理。

<img src="https://user-images.githubusercontent.com/35750880/112947010-d3255e80-9168-11eb-8aa6-b2f057761294.png" height = "300" alt="图片名称" align=center />
<img src="https://user-images.githubusercontent.com/35750880/112947020-d7517c00-9168-11eb-9fe9-4473b15158d8.png" height = "300" alt="图片名称" align=center />

## STW
在 "Stop the World" 阶段， 当前运行的所有程序将被暂停， 扫描内存的 root 节点和添加写屏障 (write barrier)。

这个阶段的第一步， 是抢占所有正在运行的 goroutine，被抢占之后， 这些 goroutine 会被悬停在一个相对安全的状态。
<img src="https://user-images.githubusercontent.com/35750880/112947028-dc163000-9168-11eb-9a03-c89bc22d8c4c.png" height = "300" alt="图片名称" align=center />


处理器 P (无论是正在运行代码的处理器还是已在 idle 列表中的处理器)， 都会被被标记成停止状态 (stopped)， 不再运行任何代码。 调度器把每个处理器的 M  从各自对应的处理器 P 分离出来， 放到 idle 列表中去。

对于 Goroutine 本身， 他们会被放到一个全局队列中等待。

<img src="https://user-images.githubusercontent.com/35750880/112947034-e0424d80-9168-11eb-86df-baae66d01fe4.png" height = "300" alt="图片名称" align=center />

### Pacing
运行时中有 GC Percentage 的配置选项，默认情况下为100。
此值表示在下一次垃圾收集必须启动之前可以分配多少新内存的比率。将 GC 百分比设置为100意味着，基于在垃圾收集完成后标记为活动的堆内存量，下次垃圾收集前，堆内存使用可以增加100%。
如果超过2分钟没有触发，会强制触发 GC。
