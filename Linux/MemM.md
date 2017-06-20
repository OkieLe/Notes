# Mem Management

[来源](http://blog.csdn.net/moxiaomomo/article/details/17104967)

内存管理子系统是操作系统的重要部分。从计算机发展早期开始，就存在对于大于系统中物理能力的内存需要。为了克服这种限制，开发了许多种策略，其中最成功的就是虚拟内存。虚拟内存通过在竞争进程之间共享内存的方式使系统显得拥有比实际更多的内存。

虚拟内存不仅仅让你的计算机内存显得更多，内存管理子系统还提供：

- Large Address Spaces（巨大的地址空间）操作系统使系统显得拥有比实际更大量的内存。虚拟内存可以比系统中的物理内存大许多倍。
- Protection（保护）系统中的每一个进程都有自己的虚拟地址空间。这些虚拟的地址空间是相互完全分离的，所以运行一个应用程序的进程不会影响另外的进程。另外，硬件的虚拟内存机制允许对内存区写保护。这可以防止代码和数据被恶意的程序覆盖。
- Memory Mapping（内存映射）内存映射用来将映像和数据映射到进程的地址空间。用内存映射，文件的内容被直接连结到进程的虚拟地址空间。
- Fair Physics Memory Allocation（公平分配物理内存）内存管理子系统允许系统中每一个运行中的进程公平地共享系统的物理内存。
- Shared Virtual Memory（共享虚拟内存）虽然虚拟内存允许进程拥有分离（虚拟）的地址空间，有时你也需要进程之间共享内存。例如，系统中可能有多个进程运行命令解释程序bash。虽然可以在每一个进程的虚拟地址空间都拥有一份bash的拷贝，更好的是在物理内存中只拥有一份拷贝，所有运行bash的进程共享代码。动态连接库是多个进程共享执行代码的另一个常见例子。共享内存也可以用于进程间通讯(IPC)机制，两个或多个进程可以通过共同拥有的内存交换信息。Linux系统支持System V的共享内存IPC机制。

##  虚拟内存的抽象模型

在进程执行程序的时候，它从内存中读取指令并进行解码。解码指令也许需要读取或者存储内存特定位置的内容，然后进程执行指令并转移到程序中的下一条指令。进程不管是读取指令还是存取数据都要访问内存。

在一个虚拟内存系统中，所有的地址都是虚拟地址而非物理地址。处理器通过操作系统保存的一组信息将虚拟地址转换为物理地址。

为了让这种转换更简单，将虚拟内存和物理内存分为适当大小的块，叫做页（page）。页的大小一样（当然可以不一样，但是这样一来系统管理起来比较困难）。Linux在Intel x86系统上使用4K字节的页。每一页都赋予一个唯一编号：page frame number(PFN 页编号)。在这种分页模型下，虚拟地址由两部分组成：虚拟页号和页内偏移量。假如页大小是4K，则虚拟地址的位11到0包括页内偏移量，位12和以上的位是页编号。每一次处理器遇到虚拟地址，它必须提取出偏移和虚拟页编号。处理器必须将虚拟页编号转换到物理的页，并访问物理页的正确偏移处。为此，处理器使用了页表（page tables）。

每一个进程拥有自己的页表。这些页表将每一个进程的虚拟页映射到内存的物理页上。页表通过虚拟页标号作为偏移来访问。虚拟页编号5是表中的第6个元素（0是第一个元素）。

要将虚拟地址转换到物理地址，处理器首先找出虚拟地址的页编号和页内偏移量。使用2的幂次的页尺寸，可以用掩码或移位简单地处理。处理器使用虚拟页编号作为索引在进程的页表中找到它的页表的条目。如果该条目有效，处理器从该条目取出物理的页编号。如果本条目无效，就是进程访问了它的虚拟内存中不存在的区域。在这种情况下，处理器无法解释地址，必须将控制权传递给操作系统来处理。

处理器具体如何通知操作系统进程在访问无法转换的无效的虚拟地址，这个方式是和处理器相关的。处理器将这种信息（page fault）进行传递，操作系统得到通知，虚拟地址出错，以及出错的原因。

假设这是一个有效的页表条目，处理器取出物理页号并乘以页大小，得到了物理内存中本页的基础地址。最后，处理器加上它需要的指令或数据的偏移量。

通过这种方式将虚拟地址映射到物理地址，虚拟内存可以用任意顺序映射到系统的物理内存中。这也导致虚拟内存的一个有趣的副产品：虚拟内存页不必按指定顺序映射到物理内存中。

### 1. Demand Paging

因为物理内存比虚拟内存少得多，操作系统必须避免无效率地使用物理内存。节省物理内存的一种方法是只加载执行程序正在使用的虚拟页。例如：一个数据库程序可能正在数据库上运行一个查询。在这种情况下，并非所有的数据必须放到内存中，而只需要正被检查的数据记录。如果这是个查找型的查询，那么加载程序中增加记录的代码就没什么意义。这种进行访问时才加载虚拟页的技术叫做demand paging。

当一个进程试图访问当前不在内存中的虚拟地址的时候处理器无法找到引用的虚拟页对应的页表条目。这时处理器通知操作系统发生page fault。

如果出错的虚拟地址无效意味着进程试图访问它不应该访问的虚拟地址。也许是程序出错，例如向内存中任意地址写。这种情况下，操作系统会中断它，从而保护系统中其他的进程。

如果出错的虚拟地址有效但是它所在的页当前不在内存中，操作系统必须从磁盘映像中将相应的页加载到内存中。相对来讲磁盘存取需要较长时间，所以进程必须等待直到该页被取到内存中。如果当前有其他系统可以运行，操作系统将选择其中一个运行。取到的页被写到一个空闲的页面，并将一个有效的虚拟页条目加到进程的页表中。然后这个进程重新运行发生内存错误的地方的机器指令。这一次虚拟内存存取进行时，处理器能够将虚拟地址转换到物理地址，所以进程得以继续运行。

Linux使用demand paging技术将可执行映像加载到进程的虚拟内存中。当一个命令执行时，包含它的文件被打开，它的内容被映射到进程的虚拟内存中。这个过程是通过修改描述进程内存映射的数据结构来实现，也叫做内存映射（memory mapping）。但是，实际上只有映像的第一部分真正放在了物理内存中。映像的其余部分仍旧在磁盘上。当映像执行时，它产生page fault，Linux使用进程的内存映像表来确定映像的那一部分需要加载到内存中执行。

### 2. Swapping（交换）

如果进程需要将虚拟页放到物理内存中而此时已经没有空闲的物理页，操作系统必须废弃物理空间中的另一页，为该页让出空间。

如果物理内存中需要废弃的页来自磁盘上的映像或者数据文件，而且没有被写过所以不需要存储，则该页被废弃。如果进程又需要该页，它可以从映像或数据文件中再次加载到内存中。

但是，如果该页已经被改变，操作系统必须保留它的内容以便以后进行访问。这种也叫做dirty page，当它从物理内存中废弃时，被存到一种叫做交换文件的特殊文件中。因为访问交换文件的速度和访问处理器以及物理内存的速度相比很慢，操作系统必须判断是将数据页写到磁盘上还是将它们保留在内存中以便下次访问。

如果决定哪些页需要废弃或者交换的算法效率不高，则会发生颠簸（thrashing）。这时，页不断地被写到磁盘上，又被读回，操作系统过于繁忙而无法执行实际的工作。进程正在使用的也叫做工作集(working set)。有效的交换方案应该保证所有进程的工作集都在物理内存中。

Linux使用LRU（Least Recently Used最近最少使用）的页面技术来公平地选择需要从系统中废弃的页面。这种方案将系统中的每一页都赋予一个年龄，这个年龄在页面存取时改变。页面访问越多，年纪越轻，越少访问，年纪越老越陈旧。陈旧的页面是交换的好候选。

### 3. Shared Vitual Memory（共享虚拟内存）

虚拟内存使多个进程可以方便地共享内存。所有的内存访问都是通过页表，每一个进程都有自己的页表。对于两个共享一个物理内存页的进程，这个物理页编号必须出现在两个进程的页表中。

这也表明了共享页的一个有趣的地方：共享的物理页不必存在共享它的进程的虚拟内存空间的同一个地方。

### 4. Physical and Vitual Addressing Modes（物理和虚拟寻址模式）

对于操作系统本身而言，运行在虚拟内存中没有什么意义。如果操作系统必须维护自身的页表，这将会是一场噩梦。大多数多用途的处理器同时支持物理地址模式和虚拟地址模式。物理寻址模式不需要页表，处理器在这种模式下不需要进行任何地址转换。Linux核心运行在物理地址模式。

### 5. Access Control（访问控制）

页表条目也包括访问控制信息。当处理器使用页表条目将进程的虚拟地址映射到物理地址的时候，它很容易利用访问控制信息控制进程不要用不允许的方式进行访问。

有很多原因你希望限制对于内存区域的访问。一些内存，比如包含执行代码，本质上是只读的代码，操作系统应该禁止进程写它的执行代码。反过来，包括数据的页可以写，但是如果试图执行这段内存应该失败。大多数处理器有两种执行状态：核心态和用户态。你不希望用户直接执行核心态的代码或者存取核心数据结构，除非处理器运行在核心态。

访问控制信息放在PTE（page table entry）中，而且和具体处理器相关。

以下两位由Linux定义并使用：
> `_PAGE_DIRTY` 如果设置，本页需要写到交换文件中。
> `_PAGE_ACCESSED` Linux 使用，标志一页已经访问过

## Caches（高速缓存）

以上理论模型可以工作，但是不会太高效率。操作系统和处理器的设计师都尽力让系统性能更高。除了使用更快的处理器、内存等，最好的方法是维护有用信息和数据的高速缓存，这会使一些操作更快。Linux使用了一系列和高速缓存相关的内存管理技术：

- Buffer cache 包含了用于块设备驱动程序的数据缓冲区。这些缓冲区大小固定（例如512字节），包括从块设备读出的数据或者要写到块设备的数据。块设备是只能通过读写固定大小的数据块来访问的设备。所有的硬盘都是块设备。块设备用设备标识符和要访问的数据块编号作为索引，用来快速定位数据块。块设备只能通过buffer cache存取。如果数据可以在buffer cache中找到，那就不需要从物理块设备如硬盘上读取，从而使访问加快。

> 参见fs/buffer.c

- Page Cache 用来加快对磁盘上映像和数据的访问。它用于缓存文件的逻辑内容，一次一页，并通过文件和文件内的偏移来访问。当数据页从磁盘读到内存中时，被缓存到page cache中。

> 参见mm/filemap.c

- Swap Cache 只有改动过的（或脏dirty）页才存在交换文件中。只要它们写到交换文件之后没有再次修改，下一次这些页需要交换出来的时候，就不需要再写到交换文件中，因为该页已经在交换文件中了，直接废弃该页就可以了。在一个交换比较厉害的系统，这会节省许多不必要和高代价的磁盘操作。

> 参见mm/swap_state.c mm/swapfile.c

- Hardware Cache 硬件高速缓存的常见的实现方法是在处理器里面：PTE的高速缓存。这种情况下，处理器不需要总是直接读页表，而在需要时把页转换表放在缓存区里。CPU里有转换表缓冲区(TLB Translation Look-aside Buffers)，放置了系统中一个或多个进程的页表条目的缓存的拷贝。

当引用虚拟地址时，处理区试图在TLB中寻找。如果找到了，它就直接将虚拟地址转换到物理地址，进而对数据执行正确的操作。如果找不到，它就需要操作系统的帮助。它用信号通知操作系统，发生了TLB missing。一个和系统相关的机制将这个异常转到操作系统相应的代码来处理。操作系统为这个地址映射生成新的TLB条目。当异常清除之后，处理器再次尝试转换虚拟地址，这一次将会成功因为TLB中该地址有了一个有效的条目。

高速缓存的副作用（不管是硬件或其他方式的）在于Linux必须花大量时间和空间来维护这些高速缓存区，如果这些高速缓存区崩溃，系统也会崩溃。

### buffers与cached的异同

buffers与cached都是内存操作，用来保存系统曾经打开过的文件以及文件属性信息，这样当操作系统需要读取某些文件时，会首先在buffers 与cached内存区查找，如果找到，直接读出传送给应用程序，如果没有找到需要数据，才从磁盘读取，这就是操作系统的缓存机制，通过缓存，大大提高了操作系统的性能。但buffers与cached缓冲的内容却是不同的。

buffers是用来缓冲块设备做的，它只记录文件系统的元数据（metadata）以及 tracking in-flight pages，而cached是用来给文件做缓冲。更通俗一点说：buffers主要用来存放目录里面有什么内容，文件的属性以及权限等等。而cached直接用来记忆我们打开过的文件和程序。

为了验证我们的结论是否正确，可以通过vi打开一个非常大的文件，看看cached的变化，然后再次vi这个文件，感觉一下两次打开的速度有何异同，是不是第二次打开的速度明显快于第一次呢？
接着执行下面的命令：
```
find /* -name  *.conf
```
看看buffers的值是否变化，然后重复执行find命令，看看两次显示速度有何不同。

## Linux Page Tables（Linux页表）

Linux假定了三级页表。访问的每一个页表包括了下一级页表的页编号。为了将虚拟地址转换为物理地址，处理器必须取得每一级字段的内容，转换为包括该页表的物理页内的偏移，然后读取下一级页表的页编号。重复三次直到包括虚拟地址的物理地址的页编号找到为止。然后用虚拟地址中的最后一个字段：字节偏移量，在页内查找数据。

Linux运行的每一个平台都必须提供转换宏，让核心处理特定进程的页表。这样，核心不需要知道页表条目的具体结构或者如何组织。通过这种方式，Linux成功地使用了相同的页表处理程序用于Alpha和Intel x86处理器，其中Alpha使用三级页表，而Intel使用二级页表。

> 参见include/asm/pgtable.h

## Page Allocation and Deallocation (页的分配和回收)

系统中对于物理页有大量的需求。例如，当程序映像加载到内存中的时候，操作系统需要分配页。当程序结束执行并卸载时需要释放这些页。另外为了存放核心相关的数据结构比如页表自身，也需要物理页。这种用于分配和回收页的机制和数据结构对于维护虚拟内存子系统的效率也许是最重要的。

系统中的所有的物理页都使用mem_map数据结构来描述。这是一个mem_map_t结构的链表，在启动时进行初始化。每一个mem_map_t（容易混淆的是这个结构也被称为page 结构）结构描述系统中的一个物理页。重要的字段（至少对于内存管理而言）是：

> 参见include/linux/mm.h

- count 本页用户数目。如果本页由多个进程共享，计数器大于1。
- Age 描述本页的年龄。用于决定本页是否可以废弃或交换出去。
- Map_nr mem_map_t描述的物理页编号。

页分配代码使用free_area向量来查找空闲的页。整个缓冲管理方案用这种机制来支持。只要用了这种代码，处理器使用的页的大小和物理页的机制就可以无关。
　
每一个free_area单元包括页块的信息。数组中的第一个单元描述了单页，下一个是2页大小的块，下一个是4页大小的块，以此类推，依次向上都是2的倍数。这个链表单元用作队列的开头，有指向mem_map数组中页的数据结构的指针。空闲的页块在这里排队。Map是一个跟踪这么大小的页的分配组的位图。如果页块中的第N块空闲，则位图中的第N位置位。

### 1. Page Allocation (页分配)

> 参见mm/page_alloc.c get_free_pages()

Linux使用Buddy算法有效地分配和回收页块。页分配代码试图分配一个由一个或多个物理页组成的块。页分配使用2的幂数大小的块。这意味着可以分配1页大小，2页大小，4页大小的块，依此类推。只要系统有满足需要的足够的空闲页（nr_free_pages > min_free_pages），分配代码就会在free_area中查找满足需要大小的一个页块。Free_area中的每一个单元都有描述自身大小的页块的占用和空闲情况的位图。例如，数组中的第2个单元拥有描述4页大小的块的空闲和占用的分配图。

这个算法首先找它请求大小的内存页块。它跟踪free_area数据结构中的list单元队列中的空闲页的链表。如果请求大小的页块没有空闲，就找下一个尺寸的块（2倍于请求的大小）。继续这一过程一直到遍历了所有的free_area或者找到了空闲页块。如果找到的页块大于请求的页块，则该块将被分开成为合适大小的块。因为所有的块都是2的幂次的页数组成，所以这个分割的过程比较简单，你只需要将它平分就可以了。空闲的块则放到适当的队列，而分配的页块则返回给调用者。

### 2. Page Deallocation（页回收）

其实分配页块的过程中将大的页块分为小的页块，将会使内存更为零散。页回收的代码只要可能就把页联成大的页块。页块的大小很重要（2的幂数），因为这样才能很容易将页块组成大的页块。

只要一个页块回收，就检查它的相邻或一起的同样大小的页块是否空闲。如果是这样，就把它和新释放的页块一起组成以一个新的下一个大小的空闲页块。每一次两个内存页块组合成为更大的页块时，页回收代码都要试图将页块合并成为更大的块。这样，空闲的页块就会尽可能的大。

## Memory Mapping （内存映射）

当一个映像执行时，执行映像的内容必须放在进程的虚拟地址空间中。对于执行映像连接到的任意共享库，情况也是一样。执行文件实际并没有放到物理内存，而只是被连接到进程的虚拟内存。这样，只要运行程序引用了映像的部分，这部分映像就从执行文件中加载到内存中。这种映像和进程虚拟地址空间的连接叫做内存映射。

每一个进程的虚拟内存用一个mm_struct 数据结构表示。这包括当前执行的映像的信息（例如bash）和指向一组vm_area_struct结构的指针。每一个vm_area_struct的数据结构都描述了内存区域的起始、进程对于内存区域的访问权限和对于这段内存的操作。这些操作是一组例程，Linux用于管理这段虚拟内存。例如其中一种虚拟内存操作就是当进程试图访问这段虚拟内存时发现（通过page fault）内存不在物理内存中所必须执行的正确操作，这个操作叫nopage 操作。Linux请求把执行映像的页加载到内存中的时候用到nopage操作。

当一个执行映像映射到进程的虚拟地址空间时，产生一组vm_area_struct数据结构。每一个vm_area_struct结构表示执行映像的一部分：执行代码、初始化数据（变量）、未初始化数据等等。Linux支持一系列标准的虚拟内存操作，当vm_area_struct数据结构创建时，一组正确的虚拟内存操作就和它们关联在一起。

## Demand Paging

只要执行映像映射到进程的虚拟内存中，它就可以开始运行。因为只有映像的最开始的部分是放在物理内存中，很快就会访问到还没有放在物理内存的虚拟空间区。当进程访问没有有效页表条目的虚拟地址的时候，处理器向Linux报告page fault。Page fault描述了发生page fault的虚拟地址和内存访问类型。

Linux必须找到page fault 发生的空间区所对应的vm_area_struct数据结构（用Adelson-Velskii and Landis AVL树型结构连接在一起）。如果找不到这个虚拟地址对应的vm_area_struct结构，说明进程访问了非法的虚拟地址。Linux将向该进程发信号，发送一个SIGSEGV信号，如果进程没有处理这个信号，它就会退出。

> 参见 handle_mm_fault() in mm/memory.c

Linux然后检查page faul的类型和该虚拟内存区所允许的访问类型。如果进程用非法的方式访问内存，比如写一个它只可以读的区域，也会发出内存错的信号。

现在Linux确定page fault是合法的，它必须进行处理。Linux必须区分在交换文件和磁盘映像中的页，它用发生page fault的虚拟地址的页表条目来确定。

> 参见do_no_page() in mm/memory.c

如果该页的页表条目是无效的但非空，此页是在交换文件中。这种情况下PFN域存放了此页在交换文件（以及那一个交换文件）中的位置。

并非所有的vm_area_struct数据结构都有一整套虚拟内存操作，而且那些有特殊的内存操作的也可能没有nopage操作。因为缺省情况下，对于nopage操作，Linux会分配一个新的物理页并创建有效的页表条目。如果这一段虚拟内存有特殊的nopage操作，Linux会调用这个特殊的代码。

通常的Linux nopage操作用于对执行映像的内存映射，并使用page cache将请求的映像页加载到物理内存中。虽然在请求的页调入的物理内存中以后，进程的页表得到更新，但是也许需要必要的硬件动作来更新这些条目，特别是如果处理器使用了TLB。既然page fault得到了处理，就可以扔在一边，进程在引起虚拟内存访问错误的指令那里重新运行。

> 参见mm/filemap.c 中filemap_nopage()

##  Page Cache

Linux的page cache的作用是加速对于磁盘文件的访问。内存映射文件每一次读入一页，这些页被存放在page cache中。page cache包括一个指向mem_map_t数据结构的指针向量：page_hash_table。Linux中的每一个文件都用一个VFS inode的数据结构标示，每一个VFS I节点都是唯一的并可以完全确定唯一的一个文件。页表的索引取自VFS 的I节点号和文件中的偏移。

> 参见linux/pagemap.h

当一页的数据从内存映射文件中读出，例如当demand paging时需要放到内存中的时候，此页通过page cache中读出。如果此页在缓存中，就返回一个指向mem_map_t数据结构的指针给page fault 的处理代码。否则，此页必须从存放此文件的文件系统中加载到内存中。Linux分配物理内存并从磁盘文件中读出该页。如果可能，Linux会启动对文件下一页的读。这种单页的超前读意味着如果进程从文件中顺序读数据的话，下一页数据将会在内存中等待。

当程序映像读取和执行的时候page cache 不断增长。如果页不在需要，将从缓存中删除。比如不再被任何进程使用的映像。当Linux使用内存的时候，物理页可能不断减少，这时Linux可以减小page cache。

## Swapping out and Discarding Pages

当物理内存缺乏的时候，Linux内存管理子系统必须试图释放物理页。这个任务落在核心交换进程上（kswapd）。核心交换守护进程是一种特殊类型的进程，一个核心线程。核心线程是没有虚拟内存的进程，以核心态运行在物理地址空间。核心交换守护进程名字有一点不恰当，因为它不仅仅是将页交换到系统交换文件上。它的任务是保证系统有足够的空闲页，使内存管理系统有效地运行。

核心交换守护进程（kswapd）在启动时由核心的init 进程启动，并等待核心的交换计时器到期。每一次计时器到期，交换进程检查系统中的空闲页数是否太少。它使用两个变量：free_pages_high和free_pages_low来决定是否释放一些页。只要系统中的空闲页数保持在free_pages_high之上，交换进程什么都不做。它重新睡眠直到它的计时器下一次到期。为了做这种检查，交换进程要考虑正在向交换文件中写的页数，用nr_async_pages来计数：每一次一页排到队列中等待写到交换文件中的时候增加，写完的时候减少。free_page_low和free_page_high是系统启动时间设置的，和系统中的物理页数相关。如果系统中的空闲页数小于free_pages_high或者比free_page_low还低，核心交换进程会尝试三种方法来减少系统使用的物理页数：

> 参见mm/vmscan.c 中的kswapd()

- 减少buffer cache 和page cache的大小
- 将System V的共享内存页交换出去
- 交换和废弃页

如果系统中的空闲页数低于free_pages_low，核心交换进程将试图在下一次运行前释放6页，否则试图释放3页。以上的每一种方法都要尝试直到释放了足够的页。核心交换进程记录了它上一次使用的释放物理页的方法。每一次运行时它都会首先尝试上一次成功的方法来释放页。

释放了足够的页之后，交换进程又一次睡眠，直到它的计时器又一次过期。如果核心交换进程释放页的原因是系统空闲页的数量少于free_pages_low，它只睡眠平时的一半时间。只要空闲页数大于free_pages_low，交换进程就恢复原来的时间间隔进行检查。

### 1. Reducing the size of the Page and Buffer Caches

page 和buffer cache中的页是释放到free_area向量中的好选择。Page Cache包含了内存映射文件的页，可能有不必要的数据，占去了系统的内存。同样，Buffer Cache 包括了从物理设备读或向物理设备写的数据，也可能包含了无用的缓冲。当系统中的物理页将要耗尽的时候，废弃这些缓存区中的页相对比较容易，因为它不需要向物理设备写（不象将页从内存中交换出去）。废弃这些页不会产生多少有害的副作用，只不过使访问物理设备和内存映射文件时慢一点。虽然如此，如果公平地废弃这些缓存区中的页，所有的进程受到的影响就是平等的。

每一次当核心交换进程要缩小这些缓存区时，它要检查mem_map页矢量中的页块，看是否可以从物理内存中废弃。如果系统空闲页太低（比较危险时）而核心交换进程交换比较厉害，这个检查的页块大小就会更大一些。页块的大小进行循环检查：每一次试图减少内存映射时都用一个不同的页块大小。这叫做clock算法，就象钟的时针。整个mem_map页向量都被检查，每次一些页。

> 参见mm/filemap.c shrink_map()

检查的每一页都要判断缓存在page cache 或者buffer cache中。注意共享页的废弃这时不考虑，一页不会同时在两个缓存中。如果该页不在这两个缓冲区中，则mem_map页向量表的下一页被检查。

缓存在buffer cache中的页（或者说页中的缓冲区被缓存）使缓冲区的分配和释放更有效。缩小内存映射的代码试图释放包含检查过的页的缓冲区。如果缓冲区释放了，则包含缓冲区的页也被释放了。如果检查的页是在Linux的page cache 中，它将从page cache 中删除并释放。

> 参见 fs/buffer.c free_buffer()

如果这次尝试释放了足够的页，核心交换进程就会继续等待直到下一次被周期性地唤醒。因为释放的页不属于任何进程的虚拟内存（只是缓存的页），因此不需要更新进程的页表。如果废弃的缓存页仍然不够，交换进程会试图交换出一些共享页。

### 2. Swapping Out System V Shared Memory Pages

System V的共享内存是一种进程间通讯的机制，通过两个或多个进程共享虚拟内存交换信息。每一块系统V共享内存都用一个shmid_ds的数据结构描述。它包括一个指向vm_area_struct链表数据结构的指针，用于共享此内存的每一个进程。vm_area_struct数据结构描述了此共享内存在每一个进程中的位置。这个系统V的内存中的每一个vm_area_struct结构都用vm_next_shared和vm_prev_shared指针连接在一起。每一个shmid_ds数据结构都有一个页表条目的链表，每一个条目都描述一个共享的虚拟页和物理页的对应关系。

核心交换进程将系统V的共享内存页交换出去时也用clock算法。它每一次运行都记录了上一次交换出去了那一块共享内存的那一页。它用两个索引来记录：第一个是shmid_ds数据结构数组中的索引，第二个是这块共享内存区的页表链中的索引。这样可以共享内存区的牺牲比较公平。

> 参见ipc/shm.c shm_swap()

因为一个指定的系统V共享内存的虚拟页对应的物理页号包含在每一个共享这块虚拟内存的进程的页表中，所以核心交换进程必须修改所有的进程的页表来体现此页已经不在内存而在交换文件中。对于每一个交换出去的共享页，交换进程必须找到在每一个共享进程的页表中对应的此页的条目（通过查找每一个vm_area_struct指针）如果在一个进程页表中此共享内存页的条目有效，交换进程要把它变为无效，并且标记是交换页，同时将此共享页的在用数减1。交换出去的系统V共享页表的格式包括一个在shmid_ds数据结构组中的索引和在此共享内存区中页表条目的索引。

如果所有共享的内存都修改过，页的在用数变为0，这个共享页就可以写到交换文件中。这个系统V共享内存区的shmid_ds数据结构指向的页表中此页的条目将会换成交换出的页表条目。交换出的页表条目无效但是包含一个指向打开的交换文件的索引和此页在此文件内的偏移量。这个信息用于将此页再取回物理内存中。

### 3. Swapping Out and Discarding Pages

交换进程轮流检查系统中的每一个进程是否可以用于交换。好的候选是可以交换的进程（有一些不行）并且有可以从内存中交换出去或废弃的一个或多个页。只有其他方法都不行的时候才会把页从物理内存交换到系统交换文件中。

> 参见 mm/vmscan.c swap_out()

映像文件的执行映像大部分内容可以从文件中重新读出来。例如：一个映像的执行指令不会被自身改变，所以不需要写到交换文件中。这些页只是被简单地废弃。如果再次被进程引用，可以从执行映像再次加载到内存中。

一旦要交换的进程确定下来，交换进程就查看它的所有虚拟内存区域，寻找没有共享或锁定的区域。Linux不会把选定进程的所有可以交换出去的页都交换出去，而只是去掉少量的页。如果页在内存中锁定，则不能被交换或废弃。

>参见mm/vmscan.c swap_out_vme() 跟踪进程mm_struct中排列的vm_area_struct结构中的vm_next　vm_nex指针。

Linux的交换算法使用了页的年龄。每一个页都有一个计数器（放在mem_map_t数据结构中），告诉核心交换进程此页是否值得交换出去。页不用时变老，访问时更新。交换进程只交换老的页。页默认第一次分配时年龄赋值为3。每一次访问，它的年龄就增加3，直到20。每一次系统交换进程运行时它将页的年龄减1使页变老。这个缺省的行为可以更改，所以这些信息（和其他相关信息）都存放在swap_control数据结构中。

如果页太老(年龄age = 0)，交换进程会进一步处理。脏页可以交换出去，Linux在描述此页的PTE中用一个和体系结构相关的位来描述这种页。但是并非所有的脏页都需要写到交换文件。每一个进程的虚拟内存区域都可以拥有自己的交换操作（由vm_area_struct中的vm_ops指针指示），如果这样，交换进程会用它的这种方式。否则，交换进程会从交换文件中分配一页，并把此页写到该文件中。

此页的页表条目会用一个无效的条目替换，但是包括了此页在交换文件的信息：此页所在文件内的偏移和所用的交换文件。不管什么方式交换，原来的物理页被放回到free_area重释放。干净（或不脏）的页可以被废弃，放回到free_area中重用。

如果交换或废弃了足够的可交换进程的页，交换进程重新睡眠。下一次唤醒时它会考虑系统中的下一个进程。这样，交换进程轻咬去每一个进程的物理页，直到系统重新达到平衡。这种做法比交换出整个进程更公平。

## Swap Cache

当把页交换到交换文件时，Linux会避免写不必要写的页。有时可能一个页同时存在于交换文件和物理内存中。这发生于一页被交换出内存然后在进程要访问时又被调入内存的情况下。只要内存中的页没有被写过，交换文件中的拷贝就继续有效。

Linux用swap cache来记录这些页。交换缓存是一个页表条目或者系统物理页的链表。一个交换页有一个页表条目，描述使用的交换文件和它在交换文件中的位置。如果交换缓存条目非0，表示在交换文件中的一页没有被改动。如果此页后来被改动了（被写），它的条目就从交换缓存中删除。

当Linux需要交换一个物理页到交换文件的时候，它查看交换缓存，如果有此页的有效条目，它不需要把此页写到交换文件。因为内存中的此页从上次读到交换文件之后没有被修改过。

交换缓存中的条目是曾经交换出去的页表条目。它们被标记为无效，但是包含了允许Linux找到正确交换文件和交换文件中正确页的信息。

## Swapping Page In

保存在交换文件中的脏页可能又需要访问。例如：当应用程序要向虚拟内存中写数据，而此页对应的物理页交换到了交换文件时。访问不在物理内存的虚拟内存页会引发page fault。Page fault是处理器通知操作系统它不能将虚拟内存转换到物理内存的信号。因为交换出去后虚拟内存中描述此页的页表条目被标记为无效。处理器无法处理虚拟地址到物理地址的转换，将控制转回到操作系统，告诉它发生错误的虚拟地址和错误的原因。这个信息的格式和处理器如何把控制转回到操作系统是和处理器类型相关的。处理器相关的page faule处理代码必须定位描述包括出错虚拟地址的虚拟内存区的vm_area_struct的数据结构。它通过查找该进程的vm_area_struct数据结构，直到找到包含了出错的虚拟地址的那一个。这是对时间要求非常严格的代码，所以一个进程的vm_area_struct数据结构按照特定的方式排列，使这种查找花费时间尽量少。

> 参见 arch/i386/mm/fault.c do_page_fault()

执行了合适的和处理器相关的动作并找到了包括错误（发生）的虚拟地址的有效的虚拟内存，page fault的处理过程又成为通用的，并可用于Linux能运行的所有处理器。通用的page fault处理代码查找错误虚拟地址的页表条目。如果它找到的页表条目是交换出去的页，Linux必须把此页交换回物理内存。交换出去的页的页表条目的格式和处理器相关，但是所有的处理器都将这些页标为无效并在页表条目中放进了在交换文件中定位页的必要信息。Linux使用这种信息把此页调回到物理内存中。

> 参见mm/memory.c do_no_page()

这时，Linux知道了错误（发生）的虚拟地址和关于此页交换到哪里去的页表条目。vm_area_struct数据结构可能包括一个例程的指针，用于把这块虚拟内存中的页交换回到物理内存中。这是swapin操作。如果这块内存中有swapin操作，Linux会使用它。其实，交换出去的系统V的共享内存之所以需要特殊的处理因为交换的系统V的共享内存页的格式和普通交换页的不同。如果没有swapin操作，Linux假定这是一个普通页，不需要特殊的处理。它分配一块空闲的物理页并将交换出去的页从交换文件中读进来。关于从交换文件哪里（和哪一个交换文件）的信息取自无效的页表条目。

> 参见mm/page_alloc.c swap_in()

如果引起page fault的访问不是写访问，页就留在交换缓存中，它的页表条目标记为不可写。如果后来此页又被写，会产生另一个page fault，这时，此页被标志为脏页，而它的条目也从交换缓存中删除。如果此页没有被修改而又需要交换出来，Linux就可以避免将此页写到交换文件，因为此页已经在交换文件中了。

如果将此页从交换文件调回的访问是写访问，这个页就从交换缓存中删除，此页的页表条目页标记为脏页和可写。

## 内存的监控

通过监控有助于了解内存的使用状态，比如内存占用是否正常，内存是否紧缺等等，监控内存最常使用的命令有free、top等，下面是某个系统free的输出：

```
[root@linuxeye ~]# free
             total       used       free     shared    buffers     cached
Mem:       3894036    3473544     420492          0      72972    1332348
-/+ buffers/cache:    2068224    1825812
Swap:      4095992     906036    3189956
```

第一行：
>total：物理内存的总大小
>used：已经使用的物理内存大小
>free：空闲的物理内存大小
>shared：多个进程共享的内存大小
>buffers/cached：磁盘缓存的大小

第二行
>Mem：代表物理内存使用情况

第三行
>(-/+ buffers/cached)：代表磁盘缓存使用状态

第四行
>Swap表示交换空间内存使用状态

**free命令输出的内存状态，可以通过两个角度来查看**

- 从内核的角度来查看内存的状态

就是内核目前可以直接分配到，不需要额外的操作，即为上面free命令输出中第二行Mem项的值，可以看出，此系统物理内存有3894036K，空闲的内存只有420492K，也就是40M多一点，我们来做一个这样的计算：
`3894036 - 3473544 = 420492`
其实就是总的物理内存减去已经使用的物理内存得到的就是空闲的物理内存大小，注意这里的可用内存值420492并不包含处于buffers和cached状态的内存大小。
如果你认为这个系统空闲内存太小，那你就错了，实际上，内核完全控制着内存的使用情况，Linux会在需要内存的时候，或在系统运行逐步推进时，将buffers和cached状态的内存变为free状态的内存，以供系统使用。

- 从应用层的角度来看系统内存的使用状态

也就是Linux上运行的应用程序可以使用的内存大小，即free命令第三行 -/+ buffers/cached 的输出，可以看到，此系统已经使用的内存才2068224K，而空闲的内存达到1825812K，继续做这样一个计算：
`420492＋（72972＋1332348）＝1825812`
通过这个等式可知，应用程序可用的物理内存值是Mem项的free值加上buffers和cached值之和，也就是说，这个free值是包括buffers和cached项大小的，对于应用程序来说，buffers/cached占有的内存是可用的，因为buffers/cached是为了提高文件读取的性能，当应用程序需要用到内存的时候，buffers/cached会很快地被回收，以供应用程序使用。

[来源：csdn](http://blog.csdn.net/moxiaomomo/article/category/831415)