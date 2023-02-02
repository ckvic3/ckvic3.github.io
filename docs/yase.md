# YASE a simple 

## Record

数据存储的基本单位是record，即一个记录。在系统中我们使用字符串来模拟。


## RID

用来指明record 的具体存储位置

// Structure of the RID:

// ---16 bits---|---24 bits---|---24 bits---|

//    File ID   |   Page Num  |   Slot Num  |


## page

人工规定的底层I/O的基本单位, 即数据在系统中的存储是按照页来存储的。page 的大小通过PAGE_SIZE 来指定，一般为4096 byte, 即4KB。用于磁盘持久化存储

为了方便存储我们设置了两种page：

### 1、data page

专门用来存储Record数据。头部为数据，尾部使用bit map 记录某个位置上是否存在数据。还包括record的长度和数量。

### 2、directory page

用来管理 data page 的状态，相当于目录page. 避免为了了解data page 的情况而要读取所有的data page


## BaseFile
提供读取磁盘文件的接口，内部最小存储单元为一个page, 类似与物理层，存取时不区分data page 和 directory page

提供以下方法：

刷新页数据 bool FlushPage(PageId pid, void *page); 即update

读取页数据 bool LoadPage(PageId pid, void *out_buf);

创建新页  PageId CreatePage();


## FILE 

建立在BaseFile的基础上，抽象化。
每个File 包含两个BaseFile，分别存储data page和 directory page.


主要就是两个方法：

请求分配页 AllocatePage()

两种情况，一种是存在之前已经被删除过的页。这个时候直接选取其中一个返回，改变状态。（快速，无磁盘操作）
另外一种就是调用BaseFILE的CreatePage()函数，在磁盘上新建一个页。（有磁盘操作）
上述两种情况都会对Directory File 的修改。

请求释放页 DeallocatePage()
删除时,修改对应的文件目录结构，将其置为未分配状态。

## Buffer manager

这里也定义了一种内存page 的类，主要是方便对于page 的抽象管理，其中利用锁来保证对于每一个page的互斥访问。
初始分配内存，作为磁盘的缓存，负责缓存从磁盘读取的页数据。

每次访问data page 或者 directory page 时，都是先从buffer 中查找，找不到才会调用BaseFile的LoadPage方法从磁盘读取数据。

对于缓存的置换策略，采用的是LRU 策略。 通过一个链表结构来实现。 

并且采用了延迟删除策略，只用当缓存已满，且target页不在缓存时，才从缓存中删除旧页。


## Index

使用跳表来实现, key 是我们从record中选取的部分区域，value 为RID，RID可以指明record 存储的文件号，页号和页内偏移。

跳表是一个多层链表，有的节点具有多层。每一层都是有序排列。跳表的时间复杂度为O(log n)


## Lock

引入事务。事务是指数据库中一定操作序列的集合。为了完成该事务一般需要访问多个records。而为了让事务能够并行完成，我们一般需要进行加锁。
这里我们简单设计了三种请求模式，

排他锁
共享锁
不加锁

lockTable 记录每个records当前处于锁的状态，并且还有其对应的访问该records的事务请求

主要功能为两个函数

bool AcquireLock(Transaction *tx, RID &rid, LockRequest::Mode mode)
事务tx请求对于rid 记录的 mode 模型锁。

请求锁时，如果该records还没有被其他事务所请求，那么就新建一个该records的锁，并且把锁分配给事务。
另一种情况，是该records已经处于加锁状态，这时我们可以观察事务是否已经获得过该锁，如果已经获得，便可以返回true.
如果未获得，那就需要与该records 锁对应列表中事务的请求锁进行相容性判断。
比如都是共享锁，那新事务就可以获得该锁。否则就不能获得锁。

当相容性不符时，我们设置了两种策略
1、no wait ，不能获得锁就立即返回false， 当前事务可能会终止
2、wait-die ，如果当前事务的timestamp 小于 之前列表中的所有事务，说明我当前事务优先级更高，因此我们使用循环请求直到当前事务获得锁。

bool ReleaseLock(Transaction *tx, RID &rid)
对于事务tx，释放其在record rid上的锁。

释放后，我们需要遍历请求列表，获得新的锁的状态。之后检查列表看是否可以为某些等待锁的事务分配锁。



