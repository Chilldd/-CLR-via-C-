<h2>21 托管堆和垃圾回收</h2>



<h3>托管堆基础</h3>

> 每个程序都要使用这样或那样的资源，包括文件、内存缓冲区、屏幕空间、网络连接、数据库资源等。事实上，在面向对象的环境中，**每个类型都代表可供程序使用的一种资源**。要使用这些资源，必须为代表资源的类型**分配内存**。以下是访问一个资源所需的步骤。
>
> 1. 调用IL指令newobj，为代表资源的类型分配内存(一般使用C#new操作符来完成)。
> 2. 初始化内存，设置资源的初始状态并使资源可用。类型的实例构造器负责设置初始状态。
> 3. 访问类型的成员来使用资源(有必要可以重复)。
> 4. 摧毁 资源的状态以进行清理。
> 5. 释放内存。垃圾回收器独自负责这一步。
>
> 如果需要程序员手动管理内存(例如，原生C++开发人员就是这样的)，这个看似简单的模式就会成为导致大量编程错误的“元凶”之一。想想看，有多少次程序员忘记释放不再需要的内存而造成内存泄漏?又有多少次试图使用已经释放的内存，然后由于内存被破坏而造成程序错误和安全漏洞?而且，这两种bug比其他大多数bug都要严重，因为一般无法预测它们的后果或发生的时间”。如果是其他bug，一旦发现程序行为异常，改正出问题的代码就行了。
>
> 现在，只要写的是可验证的、类型安全的代码(不要用c# unsafe关键字)，应用程序就不可能会出现内存被破坏的情况。内存仍有可能泄漏，但不像以前那样是默认行为。现在内存泄漏一般是因为在集合中存储了对象，但不需要对象的时候一直不去删除。
>
> 为了进一步简化编程，开发人员经常使用的大多数类型都不需要步骤4(摧毁资源的状态以进行清理)。所以，托管堆除了能避免前面提到的bug, 还能为开发人员提供一个简化的编程模型: 分配并初始化资源并直接使用。大多数类型都无需资源清理，垃圾回收器会自动释放内存。
>
> 使用需要特殊清理的类型时，编程模型还是像刚才描述的那样简单。只是有时需要尽快清理资源，而不是非要等着GC介入。可在这些类中调用-一个额外的方法(称为Dispose), 按照自己的节奏清理资源。另一方面，实现这样的类需要考虑到较多的问题(21.4节会详细讨论)。一般只有包装了本机资源(文件、套接字和数据库连接等)的类型才需要特殊清理。

<h3>从托管堆中分配资源</h3>

> **CLR要求所有对象都从托管堆分配。进程初始化时，CLR划出一个地址空间区域作为托管堆。CLR还要维护一个指针，我把它称作NextObjPtr。该指针指向下一个对象在堆中的分配位置。刚开始的时候，`NextObjPtr` 设为地址空间区域的基地址。**
>
> 一个区域被非垃圾对象填满后，CLR会分配更多的区域。这个过程一直重复，直至整个进程地址空间都被填满。所以，你的应用程序的内存受进程的虚拟地址空间的限制。32位进程最多能分配1.5GB，64 位进程最多能分配8TB。
>
>  C#的new操作符导致CLR执行以下步骤: 
>
> 1. 计算类型的字段(以及从基类型继承的字段)所需的字节数。
> 2. 加上对象的开销所需的字节数。每个对象都有两个开销字段:类型对象指针和同步块索引。对于32位应用程序，这两个字段各自需要32位，所以每个对象要增加8字节。对于64位应用程序，这两个字段各自需要64位，所以每个对象要增加16字节。CLR检查区域中是否有分配对象所需的字节数。如果托管堆有足够的可用空间，**就在NextObjPtr指针指向的地址处放入对象，为对象分配的字节会被清零**。接着调用类型的构造器(为this参数传递NextObjPtr)，
>    new操作符返回对象引用。**就在返回这个引用之前，NextObjPtr 指针的值会加上对象占用的字节数来得到一个新值，即下个对象放入托管堆时的地址**。

<img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/cb0bd160873445a7b06ae17643142dca.png?raw=true =" width="700px" />

> 图21-1 展示了包含三个对象(A，B和C)的一一个托管堆。如果要分配新对象，它将放在NextObjPtr指针指向的位置(紧接在对象C后)。
>
> 对于托管堆，分配对象只需在指针上加一个值一速度 相当快。在许多应用程序中，差不多同时分配的对象彼此间有较强的联系，而且经常差不多在同一时间访问。例如，经常在分配一个BinaryWriter 对象之前分配一个FileStream 对象。然后，应用程序使用BinaryWriter对象，而后者在内部使用FileStream对象。由于托管堆在内存中连续分配这些对象，所以会因为引用的“局部化”(locality)而获得性能上的提升。具体地说，这意味着进程的工作集会非常小，应用程序只需使用很少的内存，从而提高了速度。还意味着代码使用的对象可以全部驻留在CPU的缓存中。结果是应用程序能以惊人的速度访问这些对象，因为CPU在执行大多数操作时，不会因为“缓存未命中”(cachemiss)而被迫访问较慢的RAM。
>
> 根据前面的描述，似乎托管堆的性能天下无敌。但先别激动，刚才说的有一个大前提一内存无限，CLR总是能分配新对象。但内存不可能无限，所以CLR通过称为“垃圾回收”(GC)的技术“删除”堆中你的应用程序不再需要的对象。

> 以上摘自《CLR via C#》第四百四十七页至四百四十九页

<h3>垃圾回收算法</h3>

> 应用程序调用new操作符创建对象时，可能没有足够地址空间来分配该对象。发现空间不够，CLR就执行垃圾回收。
>
> > 前面的描述过于简单。事实上，垃圾回收是在第0代满的时候发生的。本章后面会解释“代”。在此之前，先假设堆满就发生垃圾回收。
>
> 至于对象生存期的管理，有的系统采用的是某种**引用计数算法**。事实上，Microsoft 自己的“组件对象模型”(Component Object Model, COM)用的就是引用计数。在这种系统中，堆上的每个对象都维护着一个**内存字段来统计程序中多少“部分”正在使用对象**。随着每一“部分”到达代码中某个不再需要对象的地方，就递减对象的计数字段。计数字段变成0对象就可以从内存中删除了。许多引用计数系统最大的问题是**处理不好循环引用**。例如在GUI应用程序中，窗口将容纳对子Ul元素的引用，而子UI元素将容纳对父窗口的引用。这种引用会阻止两个对象的计数器达到0，所以两个对象永远不会删除，即使应用程序本身不再需要窗口了。
>
> 鉴于引用计数垃圾回收器算法存在的问题，CLR改为使用一种**引用跟踪算法**。**引用跟踪算法只关心引用类型的变量**，因为只有这种变量才能引用堆上的对象;值类型变量直接包含值类型实例。引用类型变量可在许多场合使用，包括类的静态和实例字段，或者方法的参数和局部变量。我们将**所有引用类型的变量都称为根**。
> **CLR开始GC时，首先暂停进程中的所有线程**。**这样可以防止线程在CLR检查期间访问对象并更改其状态**。然后，CLR进入GC的**标记阶段**。在这个阶段，CLR**遍历堆中的所有对象**，将**同步块索引字段中的一位设为0。这表明所有对象都应删除**。然后，CLR检查所有**活动根**，**查看它们引用了哪些对象**。这正是CLR的GC称为引用跟踪GC的原因。如果一个根包含null，CLR忽略这个根并继续检查下个根。**任何根如果引用了堆上的对象，CLR都会标记那个对象，也就是将该对象的同步块索引中的位设为1**。**一个对象被标记后，CLR会检查那个对象中的根，标记它们引用的对象**。**如果发现对象已经标记，就不重新检查对象的字段。这就避免了因为循环引用而产生死循环**。
>
> > gy_note:
> >
> > GC如何计算出需要删除的对象：
> >
> > 1. 暂停所有线程
> > 2. 遍历所有对象，将对象的同步块索引第一位修改为0（0即是删除，这一步相当于把所有对象都变成待删除状态）。
> > 3. CLR检查所有根（引用类型的变量）如果根引用了堆上的对象，就把这个对象的同步块索引的第一位设置为1。
> > 4. 当一个对象的同步块索引第一位被标记成1时，检查这个对象引用的其他对象（就是看对象的字段中是否引用了其他的对象），标记它们。（如果对象已被标记就不检查对象的字段了，以防循环引用）
> >
> > 因为所有对象默认都修改成删除状态，所以只需要把正在使用的对象修改为正常状态即可。
>
> 图21-2展示了一个堆，其中包含几个对象。应用程序的根直接引用对象A，C，D和F。所有对象都已标记。标记对象D时，垃圾回收器发现这个对象含有一个引用对象H的字段，造成对象H也被标记。标记过程会持续，直至应用程序的所有根所有检查完毕。

<img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/a3f3fcac290043e091b574ec58ac4ae1.png?raw=true =" width="700px" />

>检查完毕后，堆中的对象要么已标记，要么未标记。已标记的对象不能被垃圾回收，因为至少有一个根在引用它。我们说这种对象是**可达(reachable)**的，因为应用程序代码可通过仍在引用它的变量抵达(或访问)它。未标记的对象是**不可达(unreachable)**的，因为应用程序中不存在使对象能被再次访问的根。

> CLR知道哪些对象可以幸存，哪些可以删除后，就进入GC的**压缩(compact)**阶段。在这个阶段，CLR对堆中已标记的对象进行“乾坤大挪移”，压缩所有幸存下来的对象，**使它们占用连续的内存空间**。这样做有许多好处。首先，所有幸存对象在内存中紧挨在一起，**恢复了引用的“局部化”**，减小了应用程序的工作集，从而提升了将来访问这些对象时的性能。其实，可用空间也全部是连续的，所以这个地址空间区段得到了解放，允许其他东西进驻。最后，压缩意味着托管堆解决了本机(原生)堆的空间碎片化问题。
>
> > 这里的压缩其实是指**“碎片整理”**，就是把剩余可用的对象位置整理成连续的。
>
> 在内存中移动了对象之后有一个问题待解决。引用幸存对象的根现在引用的还是对象最初在内存中的位置，而非移动之后的位置。被暂停的线程恢复执行时，将访问旧的内存位置，会造成内存损坏。这显然不能容忍的，所以作为压缩阶段的一部分，**CLR还要从每个根减去所引用的对象在内存中偏移的字节数。这样就能保证每个根还是引用和之前一样的对象; 只是对象在内存中变换了位置**。
>
> 压缩好内存后，托管堆的NextObjPtr指针指向最后一个幸存对象之后的位置。下一个分配的对象将放到这个位置。图21-3展示了压缩阶段之后的托管堆。压缩阶段完成后，CLR恢复应用程序的所有线程。这些线程继续访问对象，就好象GC没有发过一样。

<img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/385e03cb40234aba9f501d38b85497b5.png?raw=true =" width="700px" />

> 如果CLR在一次GC之后回收不了内存，而且进程中没有空间来分配新的GC区域，就说明该进程的内存已耗尽。此时，试图分配更多内存的new操作符会抛出
> OutOfMemoryException。应用程序可捕捉该异常并从中恢复。但大多数应用程序都不会这么做;相反，异常会成为未处理异常，Windows将终止进程并回收进程使用的全部内存。
>
> > 静态字段引用的对象一直存在，直到用于加载类型的AppDomain卸载为止。内存泄漏的一个常见原因就是让静态字段引用某个集合对象，然后不停地向集合添加数据项。静态字段使集合对象一直存活，而集合对象使所有数据项一直存活。因此，应尽量避免使用静态字段。

> gy_note:
>
> GC执行步骤：
>
> 1. 暂停所有线程。
> 2. 标记需要删除的对象。
> 3. 删除对象。
> 4. 压缩剩余存活对象。
> 5. 修改根引用，使其执行正确的对象。
> 6. 将托管堆的NextObjPtr指针指向最后一个幸存对象的位置。
>
> GC中还有一个重要的概念**“代”**，不过整体步骤基本不变。



<h3>垃圾回收和调试(了解)</h3>

<img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/ce6003712da64fe4aedddccf706a11f0.png?raw=true =" width="700px" />



<h3>代</h3>

> CLR的GC是基于**代**的垃圾回收器(generational garbage collector)", 它对你的代码做出了以下几点假设。
>
> 1. 对象越新，生存期越短。
> 2. 对象越老，生存期越长。
> 3. 回收堆的一部分，速度快于回收整个堆。
>
> 大量研究证明，这些假设对于现今大多数应用程序都是成立的，它们影响了垃圾回收器的实现方式。本节将解释代的工作原理。托管堆在初始化时不包含对象。添加到堆的对象称为第0代对象。简单地说，**第0代对象就是那些新构造的对象**，**垃圾回收器从未检查过它们**。图21-4展示了一个新启动的应用程
> 序，它分配了5个对象(从A到E)。过了一会儿，对象C和E变得不可达。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/c48870b9989d4a18875120e0220750c5.png?raw=true =" width="700px" />
>
> CLR初始化时为第0代对象选择一个预算容量(以KB为单位)。如果分配一个新对象造成第0代超过预算，就必须启动一次垃圾回收。假设对象A到E刚好用完第0代的空间，那么分配对象F就必须启动垃圾回收。垃圾回收器判断对象C和E是垃圾，所以会压缩对象D，使之与对象B相邻。**在垃圾回收中存活的对象(A，B和D)现在成为第1代对象**。**第1代对象已经经历了垃圾回收器的一次检查**。此时的堆如图21-5所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/0a0dc11bfc56473eb68dd4de5c019262.png?raw=true =" width="700px" />
>
> 一次垃圾回收后，第0代就不包含任何对象了。和前面一样，新对象会分配到第0代中。在图21-6中，应用程序继续运行，并新分配了对象F到对象K。另外，随着应用程序继续运行，对象B，H和J变得不可达，它们的内存将在某一时刻回收。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/8d3458c9c41e4eb19df173add1452877.png?raw=true =" width="700px" />
>
> 现在，假定分配新对象L会造成第0代超出预算，造成必须启动垃圾回收。开始垃圾回收时，垃圾回收器必须决定**检查哪些代**。前面说过，CLR初始化时会为第0代对象选择预算。事实上，它还必须为第1代选择预算。开始一次垃圾回收时，垃圾回收器还会检查第I代占用了多少内存。在本例中，由于第1
> 代占用的内存远少于预算，所以垃圾回收器只检查第0代中的对象。回顾一下基于代的垃圾回收器做出的假设。第一个假设是越新的对象活得越短。因此，第**0代包含更多垃圾的可能性很大**，能回收更多的内存。由于忽略了第1代中的对象，所以加快了垃圾回收速度。
>
> 显然，忽略第1代中的对象能提升垃圾回收器的性能。但对性能有更大提振作用的是现在不必遍历托管堆中的每个对象。**如果根或对象引用了老一代的某个对象，垃圾回收器就可以忽略老对象内部的所有引用**，能在更短的时间内构造好**可达对象图(graphofreachableobject)**。当然，老对象的字段也有可能引用新对象。为了确保对老对象的已更新字段进行检查，垃圾回收器利用了JIT 编译器内部的一个机制。这个机制在**对象的引用字段发生变化时，会设置一个对应的位标志**。这样，垃圾回收器就知道自上一次垃圾回收以来，哪些老对象(如果有的话)已被写入。只有字段发生变化的老对象才需检查是否引用了第0代中的任何新对象。
>
> > 当JIT编译器生成本机(native)代码来修改对象中的一个引用字段时，本机代码会生成对一个write barrier方法的调用(译注: write barrier方法是在有数
> > 据向对象写入时执行一些内存管理代码的机制)。这个write barrier 方法检查字段被修改的那个对象是否在第1代或第2代中。如果在，write barrier代码就在一个所 谓的card table中设置一个 bit。card table为堆中的每128字节的数据都准备好了一个bit。GC下一次启动时会扫描cardtalbe，了解第1代和第
> > 2代中的哪些对象的字段自上次GC以来已被修改。任何被修改的对象引用了第0代中的一个对象，被引用的第0代对象就会在垃圾回收过程中“存活”。GC之后，card table 中的所有bit 都被重置为0。向对象的引用字段中写入时，write barrier代码会造成少量性能损失(对应地，向局部变量或静态字段写入便不会有这个损失)。另外，如果对象在第1代或第2代中，性能会损失得稍微多一些。
>
> 基于代的垃圾回收器还假设越老的对象活得越长。也就是说，第1代对象在应用程序中很有可能是继续可达的。如果垃圾回收器检查第I代中的对象，很有可能找不到多少垃圾，结果是回收不了多少内存。因此，对第1代进行垃圾回收很可能是浪费时间。如果真的有垃圾在第1代中，它将留在那里。此时的堆如图21-7所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/4831d2c5e3da4f82a6b7520f4d442238.png?raw=true =" width="700px" />
>
> 如你所见，所有幸存下来的第0代对象都成了第1代的一部分。由于垃圾回收器没有检查
> 第1代，所以对象B的内存并没有被回收，即使它在上一次垃圾回收时已经不可达。同样，
> 在一次垃圾回收后，第0代不包含任何对象，等着分配新对象。假定应用程序继续运行，
> 并分配对象L到对象O。另外，在运行过程中，应用程序停止使用对象G，L和M，使它
> 们变得不可达。此时的托管堆如图21-8所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/871bb1a1015447a2a32ebecbf5b4a106.png?raw=true =" width="700px" />
>
> 假设分配对象P导致第0代超过预算，垃圾回收发生。由于第1代中的所有对象占据的内
> 存仍小于预算，所以垃圾回收器再次决定只回收第0代，忽略第1代中的不可达对象(对象
> B和G)。回收后，堆的情况如图21-9所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/12ee73f0e7ba4a7aac317e41658956b9.png?raw=true =" width="700px" />
>
> 从图21-9可以看到，第1代正在缓慢增长。假定第1代的增长导致它的所有对象占用了全
> 部预算。这时，应用程序继续运行(因为垃圾回收刚刚完成)，并分配对象P到对象S,使第
> 0代对象达到它的预算容量。这时的堆如图21-10所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/1a3b31fa88904a4cb26bdac21f3d37e8.png?raw=true =" width="700px" />
>
> 应用程序试图分配对象T时，由于第0代已满，所以必须开始垃圾回收。但这-次垃圾回
> 收器发现第1代占用了太多内存，以至于用完了预算。由于前几次对第0代进行回收时，
> 第1代可能已经有许多对象变得不可达(就像本例这样)。所以这次垃圾回收器决定检查第
> 1代和第0代中的所有对象。两代都被垃圾回收后，堆的情况如图21-11所示。
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/b5baf8f31dcd4ddbbaae8cb875ffed1a.png?raw=true =" width="700px" />
>
> 和之前一一样，垃圾回收后，第0代的幸存者被提升至第1代，第1代的幸存者被提升至第
> 2代，第0代再次空出来了，准备好迎接新对象的到来。第2代中的对象经过了2次或更
> 多次检查。虽然到目前为止已发生过多次垃圾回收，但只有在第1代超出预算时才会检查
> 第1代中的对象。而在此之前，一般都已经对第0代进行了好几次垃圾回收。
> 托管堆只支持三代:第0代、第1代和第2代。没有第3代。CLR初始化时，会为每一代
> 选择预算。然而，CLR的垃圾回收器是自调节的。这意味着垃圾回收器会在执行垃圾回收
> 的过程中了解应用程序的行为。例如，假定应用程序构造了许多对象，但每个对象用的时
> 间都很短。在这种情况下，对第0代的垃圾回收会回收大量内存。事实上，第0代的所有
> 对象都可能被回收。
> 如果垃圾回收器发现在回收0代后存活下来的对象很少，就可能减少第0代的预算。已
> 分配空间的减少意味着垃圾回收将更频繁地发生，但垃圾回收器每次做的事情也减少了,
> 这减小了进程的工作集。事实上，如果第0代中的所有对象都是垃圾，垃圾回收时就不必
> 压缩(移动)任何内存;只需让NextObjPtr指针指回第0代的起始处即可。这样回收可真快!
>
> 另一方面，如果垃圾回收器回收了第0代，发现还有很多对象存活，没有多少内存被回收，
> 就会增大第0代的预算。现在，垃圾回收的次数将减少，但每次进行垃圾回收时，回收的
> 内存要多得多。顺便说- -句， 如果没有回收到足够的内存，垃圾回收器会执行一次完整回
> 收。如果还是不够，就抛出OutOfMemoryException异常。
>
> 到目前为止，只是讨论了每次垃圾回收后如何动态调整第0代的预算。但垃圾回收器还会
> 用类似的启发式算法调整第1代和第2代的预算。这些代被垃圾回收时，垃圾回收器会检
> 查有多少内存被回收，以及有多少对象幸存。基于这些结果，垃圾回收器可能增大或减小
> 这些代的预算，从而提升应用程序的总体性能。最终的结果是，垃圾回收器会根据应用程
> 序要求的内存负载来自动优化一这非常“酷”!
>
> <img src="https://github.com/Chilldd/CLR_via_C_Sharp_Note/blob/main/IMG/21/b7468a147bb548199b5a3a0f7008dfd2.png?raw=true =" width="700px" />
