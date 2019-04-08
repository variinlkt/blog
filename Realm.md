# Realm
最近使用了realm来搭建毕设的数据库，本来以为它只是个普通的封装了SQLite的本地数据库，在写论文介绍Realm时偶然看到它是线程安全的。。然后忍不住多了解了一下，发现Realm真是一个很酷的东西诶
## 什么是线程安全
同步机制两个或两个以上的线程在同时操作一个共享资源时仍然能得到正确的结果,则称为线程安全

## 什么是MVCC
MVCC 在设计上采用了和 Git 一样的源文件管理算法。你可以把 Realm 的内部想象成一个 Git，它也有分支和原子化的提交操作。这意味着你可能工作在许多分支上（数据库的版本），但是你却没有一个完整的数据拷贝。Realm 和真正的 MVCC 数据库还是有些不同的。一个像 Git 的真正的 MVCC 数据库，你可以有成为版本树上 HEAD 的多个候选者。而 Realm 在某个时刻只有一个写操作，而且总是操作最新的版本 - 它不可以在老的版本上工作。

更进一步，Realm 更像一个庞大的树形数据结构（准确的说是一个 B 树），任何时候，你都有最上层的节点，如下 R 节点（和 Git 的 HEAD 提交类似）。
一旦你要提交改变，copy-on-write 才会发生。Copy-on-write 意味着你重新创建了树的另一个分支，然后在不改变现有数据的情况下完成写操作。
采用这种方法，如果在写事务过程中发生了错误的话，原始数据是不受影响的，顶指针依旧指向没有被损坏的数据，因为你是在别处做的写操作。Realm 采用了两阶段提交来验证写操作，Realm 会验证所有写到磁盘中的内容，这样数据才是安全的。只有在这个时候，Realm 的指针才会移动而且说，“好的，这是新的官方的版本。” 这意味着在一次写事件中最坏的情况是你仅仅失去你更新的数据，而不是整个 Realm 数据库。

以下内容翻译自[维基百科](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)：

> MVCC，全称Multiversion concurrency control多版本并发控制，是一个广泛用于数据库管理系统，用于提供数据库和编程语言的并发访问以此来执行事务型内存的并发控制的方法
> 
> 如果没有并发控制，如果有人想从一个数据库中读数据，同时另一个人想写数据，那么就有可能得到写了一半或不一致的结果。比如，在两个网络传输的银行账户中，一个用户在银行查看余额，与此同时里面的钱正在被取出但是还没转存到另一个账户中，这种情况会导致这笔钱会好像从银行里消失了。隔离可以提供并发读取数据的保证，它通过并发控制的协议来执行。想要让所有的读数据的人等写数据的人写完，最简单的方法是读写锁。锁带来了关于长读取事务和更新事务的争论。MVCC致力于通过保存好几份数据的拷贝来解决这个问题。通过这个方法，每个在同一时间连接到这个数据库的用户看到的都是当前数据库的快照。所有写入的数据不会被这个数据库的其他用户所看到，直到事务完成。
> 
> 当一个MVCC数据库需要更新一个数据时，它不会在原始的数据中修改，而是创建了一个数据的新版本。因此一个数据库存储了好几个版本。每个事务所看到的数据库版本依赖于事务隔离级别的执行。最常见的MVCC的事务隔离级别的执行是快照隔离级别。有了快照隔离级别，当事务开始时，这个事务会observe数据的state。MVCC介绍了怎样移除需要淘汰的并且不会再被读取的版本的难点。在一些情况下，它会执行周期性地清除和删除过期的版本的过程。这通常是一个stop-the-world的过程，贯穿了整个表并用最新的版本的数据来重写它。数据库将存储空间分为两部分：数据部分和undo log。data部分保存了最后提交的版本。undo log允许旧版本数据的更新。后者方法的主要的不足在于当有大量的更新的工作量时，undo log部分会超出运行内存，事务会因不能给出他们的快照而被终止。
> 一个document-oriented数据库也会允许系统去充分利用documents，通过把整个documents写入磁盘的相邻部分，这样一来当数据更新时，整个document就能被复写，而不是一些字节片段被截断或保存在一个连接的、非相邻的数据库结构中。
> 
> MVCC提供了时间点一致的view。MVCC下的读事务典型地使用了时间戳或事务id来确定读取数据库的什么state，并且读这些版本的数据。读和写事务也因此相互独立、没必要上锁。然而，即使锁是没必要的，锁还是被用在了一些MVCC数据库中比如Oracle。写操作会创建一个新版本，而并发的读操作读取的是一个旧版本。

翻译得真尴尬，有些还看不太懂

总结一下就是：
- MVCC是多版本并发控制，它解决了数据库并发读写操作带来的问题
- 它把数据库分为了data和undo log两部分，data部分保存了最后提交的版本。undo log允许旧版本数据的更新
- MVCC依赖版本控制，读和写事务相互独立：
    - MVCC下的读事务使用时间戳或事务id来确定读取数据库的什么state，并发的读操作读取的是一个旧版本。
    - 当update一个数据时，它会创建一个数据的新版本而不是在原数据中修改。
- MVCC还会周期性地清除过期版本 
- MVCC存在事务隔离级别，每个事务所看到的数据库版本依赖于事务隔离级别，常见的事务隔离级别是快照隔离级别。




## Snapshot isolation快照隔离级别
以下内容翻译自[维基百科](https://en.wikipedia.org/wiki/Snapshot_isolation)：
> 在数据库中和事务执行过程中，快照隔离级别是保证了一个事务里面所有的读操作可以看到一个一致的数据库快照（实际上它是读取了最新的提交），同时事务只有当没有updates操作与任何并发的updates操作有冲突时才会成功提交。



## zero-copy 架构
### 从 ORM关系型对象映射）、Core Data 中获取对象的传统方法
> 大部分的时候，你都把数据存在磁盘上的数据库文件中。开发者发起一个从持久化机制（比如 ORM 或者 Core Data）中获取数据的请求，数据格式会是和本地平台密切相关的（比如 安卓或者苹果）。
> 
> 这个时候，持久化机制会把请求转换成一系列的 SQL 语句，创建一个数据库连接（如果没有创建的话），发送到磁盘上，执行查询，读取命中查询的每一行的数据，然后存到内存里（这里有内存消耗）。之后你需要把数据序列化成可在内存里面存储的格式，这意味着比特对齐，这样 CPU 才能处理它们。最后，数据需要转换成语言层面的类型，然后它会以对象的形式返回，这样平台才能用（POJO， NSManagedObject 等等）来处理它。如果你在你的持续化机制中有子引用或者列表引用的话，这个过程会更复杂。这个过程会一遍一遍的执行（取决于你的持续化机制和配置）。

总结：
- 开发者发起一个从持久化机制中获取数据的请求
- 持久化机制会把请求转换成一系列的 SQL 语句
- 创建一个数据库连接
- 发送到磁盘上
- 执行查询
- 读取命中查询的每一行的数据
- 数据存到内存
- 把数据序列化成可在内存里面存储的格式，以便让CPU处理它们
- 数据需要转换成语言层面的类型，以便平台进行处理

可以看到这个过程非常复杂

那零拷贝架构是什么？

> Zero-copy versions of operating system elements, such as device drivers, file systems, and network protocol stacks, greatly increase the performance of certain application programs and more efficiently utilize system resources. Performance is enhanced by allowing the CPU to move on to other tasks while data copies proceed in parallel in another part of the machine. Also, zero-copy operations reduce the number of time-consuming mode switches between user space and kernel space. System resources are utilized more efficiently since using a sophisticated CPU to perform extensive copy operations, which is a relatively simple task, is wasteful if other simpler system components can do the copying.
> 
> As an example, reading a file and then sending it over a network the traditional way requires two data copies and two context switches per read/write cycle. One of those data copies uses the CPU. Sending the same file via zero copy reduces the context switches to two and eliminates all CPU data copies.[1]
> 
> Zero-copy protocols are especially important for high-speed networks in which the capacity of a network link approaches or exceeds the CPU's processing capacity. In such a case the CPU spends nearly all of its time copying transferred data, and thus becomes a bottleneck which limits the communication rate to below the link's capacity. A rule of thumb used in the industry is that roughly one CPU clock cycle is needed to process one bit of incoming data.

总结：
- 零复制操作减少了用户空间和内核空间之间耗时的模式切换次数。
- 通过允许CPU继续执行其他任务来增强性能，同时数据副本在机器的另一部分中并行进行。
- 零复制操作减少了用户空间和内核空间之间耗时的模式切换次数。 
- 零拷贝协议对于网络链路容量接近或超过CPU处理能力的高速网络尤为重要
- 读取文件然后通过网络发送传统方式需要每个读/写周期两个数据副本和两个上下文切换。 通过零拷贝发送相同的文件可将上下文切换减少到两个并消除所有CPU数据副本

## Realm的特性
Realm 跳过了整个拷贝过程，因为数据库文件是 memory-mapped。Realm 它允许文件能在没有做任何反序列化的情况下可以在内存中读取。Realm 跳过了所有这些开销很大的步骤，而这些步骤在传统的持久化机制中必须执行。**Realm 只需要简单地计算偏移来找到文件中的数据，然后从原始访问点返回数据结构(POJO/NSManagedObject/等等)的值**。这更有效而且更快。

### 数据结构
Realm 中的对象关系也特别的快，因为它们是相关对象的一个**类 B 树**的数据结构的索引。这比查询快多了。正因为如此，没有必要再进行一次像 ORM 做的全查询了。它是个简单的指向相关对象的本地指针。这就是所有要做的事情了。

### 自动更新对象和查询
零拷贝架构不仅仅提供了速度。底层数据改变了视图会自动更新，这意味着永远不需要重取数据。对象的改变会立马改变查询的结果。

### 当 Realm 数据变化时获取通知
假设以下的情况：有一个 UI 线程使用对象来显示一些 UI 值。一个后台线程通过写操作改变了这个对象。几乎是即时的（在运行循环的下个迭代），UI 线程的数据对象被更新了（记住对象可以直接脱离核心数据库工作，因为有零拷贝架构）。

后台线程通过 Realm 的改变监听者发送给 UI 线程一个通知消息说，有一个改变发生了。（这个特性在大部分的 Realm 产品中都已实现，包括 Java，Objective-C，和 Swift，React Native 正在开发中。）这个时候，UI 线程可以更新视图来显示新的数据了

### Realm 对象不能在线程间传递
因为 Realm 是基于零拷贝架构，所有对象是鲜活的而且自动更新。如果 Realm 允许对象可在线程间共享，Realm 会**无法确保数据的一致性**，因为不同的线程会在不确定的什么时间点同时改变对象的数据。这样数据很快就不一致了。一个线程可能需要写入一个数据而另一个线程也打算读取它，反过来也可能。

是的，这可以通过许多方法来解决，一个常用的方法就是锁住对象，存储器和访问器。虽然这能工作，但是锁会变成一个头疼的性能瓶颈。除了性能，锁的其他问题也很明显，因为锁 —— 一个长时间的后台写事务会阻塞 UI 的读事务。如果我们采用锁机制，我们会失去太多的 Realm 可提供的速度优势和数据一致性的保证。

#### 解决方法
如果你需要在另外一个线程中获取同样的数据，你只需要在该线程里面重新查询。或者，更好的方法是，**用 Realm 的响应式架构监听变化**！各个线程的所有对象都是自动更新的 - Realm 会在数据变化时通知你。你只需要对这些变化做出响应就可以了。 

