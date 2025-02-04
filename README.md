# 《CLR via C#》学习笔记



最近在学习《CLR via C#》第四版，这本书信息量巨大，干货过于多，所以觉得需要边学边记录一下笔记，也方便加深下印象。  

由于之前看书的时候没记录笔记，所以下面的目录可能会比笔记多，不过笔记我会慢慢补上去，直到把这本书我认为重要的内容全记下来。  

笔记内容基本是从书上摘抄下来的，难理解的部分我替换成自己理解的形式替换了上去，还有一些自己记录的笔记。

书中内容过多，所以我只记录了部分重要内容，一些比较基础的就不记录了。  

 

<h2>已学习目录(个人觉得比较重要的内容)</h2>

| **标题**                             | **页码** | **笔记**                                                     | **重要等级** |
| ------------------------------------ | -------- | ------------------------------------------------------------ | ------------ |
| 1.4  执行程序集的代码                | 10       | 1. IL是什么<br/>**2. IL如何转化为CPU指令**<br/>3. **方法被调用时，发生过程** | **高**       |
| 1.7  通用类型系统（CTS）             | 22       | CTS(Common Type System)介绍                                  | 中           |
| 1.8  公共语言规范                    | 24       | CLS(Common Language Specification)公共语言规范               | 中           |
| 2.3  元数据                          | 34       | **元数据介绍**                                               |              |
| 3.8  运行时如何解析类型引用          | 70       | **CLR如何根据元数据解析运行类型引用**                        |              |
| 4.1  所有对象都从Object派生          | 81       | 1. Object介绍<br/>**2. new操作符所做的事情**                 | **高**       |
| 4.2  类型转换                        | 83       | 对象如何进行安全的类型转换                                   | **高**       |
| 4.4  运行时的相互关系                | 90       | **1. 类型，对象，线程栈，托管堆在运行时的相互关系**<br/>2. **静态方法，实例方法，虚方法的调用区别** | **高**       |
| 5.1  基元类型                        | 99       | **基元类型介绍**                                             | **高**       |
| 5.2  引用类型，值类型                | 106      | **值类型和引用类型的区别**                                   | **高**       |
| 5.3  装箱，拆箱                      | 111      | **装箱和拆箱详解**                                           | **高**       |
| 6.4  静态类                          | 141      | **静态类**                                                   | 中           |
| 6.6.1  CLR如何调用虚方法、属性和事件 | 145      | **CLR如何调用方法**                                          | **高**       |
| 7  常量和字段                        | 155      | **1. 常量**<br/>2. **字段**<br/>3. **readonly(readonly修饰引用类型时，不可变的是引用，而不是引用的对象)** | **高**       |
| 8  方法                              | 161      | **1. 构造函数（值类型，引用类型）**<br/>2. **类型构造器**<br/>3. **操作符方法**<br/>4. **扩展方法**<br/>5. **分部方法** | **高**       |





<h2>CLR基础</h2>



<h3>CLR简介</h3>

> 公共语言运行时(Common Language Runtime, CLR)是一个可由**多种编程语言使用**的“运行时”。CLR的核心功能(比如内存管理、程序集加载、安全性、异常处理和线程同步)可**由面向CLR的所有语言使用**。例如，“运行时” 使用异常来报告错误: 因此，面向它的任何语言都能通过异常来报告错误。另外，“运行时”允许创建线程，所以面向它的任何语言都能创建线程。
>
> 事实上，在运行时，CLR根本不关心开发人员用哪一种语言写源代码。这意味着在选择编程语言时，应选择最容易表达自己意图的语言。可用任何编程语言开发代码，只要**编译器是面向CLR**的。既然如此，不同编程语言的优势何在呢? 事实上，可将编译器视为语法检查器和“正确代码”**分析器**。它们检查源代码，确定你写的一-切都有意义，并输出对你的意图进行描述的代码。不同编程语言允许用不同的语法来开发。不要低估这个选择的价值
>
> 摘自《CLR via C# 》第四页



> .NET 提供了一个称为公共语言运行时的运行时环境，它运行代码并提供使开发过程更轻松的服务。
>
> 公共语言运行时的功能通过编译器和工具公开，你可以编写利用此托管执行环境的代码。 使用**面向运行时的语言编译器开发的代码称为托管代码**。 托管代码具有许多优点，例如：**跨语言集成、跨语言异常处理、增强的安全性、版本控制和部署支持、简化的组件交互模型、调试和分析服务**等。
>
> 若要使公共语言运行时能够向托管代码提供服务，语言编译器必须生成一些**元数据**来描述代码中的类型、成员和引用。 **元数据与代码一起存储**；每个可加载的公共语言运行时可迁移执行 (PE) 文件都包含元数据。 公共语言运行时使用元数据来完成以下任务：**查找和加载类，在内存中安排实例，解析方法调用，生成本机代码，强制安全性，以及设置运行时上下文边界**。
>
> 公共语言运行时自动处理对象布局并管理对象引用，当不再使用对象时释放它们。 **按这种方式实现生存期管理的对象称为托管数据**。 垃圾回收消除了内存泄漏以及其他一些常见的编程错误。 如果你编写的代码是托管代码，则可以在 .NET 应用程序中使用托管数据、非托管数据或者同时使用这两种数据。 由于语言编译器会提供自己的类型（如基元类型），因此你可能并不总是知道（或需要知道）这些数据是否是托管的。
>
> 有了公共语言运行时，就可以很容易地设计出对象能够跨语言交互的组件和应用程序。 也就是说，用不同语言编写的对象可以互相通信，并且它们的行为可以紧密集成。 例如，可以定义一个类，然后使用不同的语言从原始类派生出另一个类或调用原始类的方法。 还可以将一个类的实例传递到用不同的语言编写的另一个类的方法。 这种跨语言集成之所以成为可能，是因为基于公共语言运行时的语言编译器和工具使用由公共语言运行时定义的常规类型系统，而且它们遵循公共语言运行时关于定义新类型以及创建、使用、保持和绑定到类型的规则。
>
> 所有托管组件都带有生成它们所基于的组件和资源的信息，这些信息构成了元数据的一部分。 公共语言运行时使用这些信息确保组件或应用程序具有它需要的所有内容的指定版本，这样就使代码不太可能由于某些未满足的依赖项而发生中断。 注册信息和状态数据不再保存在注册表中（因为在注册表中建立和维护这些信息很困难）。 取而代之的是，有关你定义的类型（及其依赖项）的信息作为元数据与代码存储在一起，这样大大降低了组件复制和移除任务的复杂性。
>
> 语言编译器和工具公开公共语言运行时的功能的方式对于开发人员来说不仅很有用，而且很直观。 这意味着，公共语言运行时的某些功能可能在一个环境中比在另一个环境中更突出。 你对公共语言运行时的体验取决于所使用的语言编译器或工具。 例如，如果你是一位 Visual Basic 开发人员，你可能会注意到：有了公共语言运行时，Visual Basic 语言的面向对象的功能比以前多了。 运行时提供如下优点：
>
> - 性能得到了改进。
> - 能够轻松使用用其他语言开发的组件。
> - 类库提供的可扩展类型。
> - 语言功能，如面向对象的编程的继承、接口和重载。
> - 允许创建多线程的可缩放应用程序的显式自由线程处理支持。
> - 结构化异常处理支持。
> - 自定义特性支持。
> - 垃圾回收。
> - 使用委托取代函数指针，从而增强了类型安全和安全性。 有关委派的详细信息，请参阅<a href="https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system" target="_blank" >通用类型系统</a>。
>
> 
>
> <a href="https://docs.microsoft.com/zh-cn/dotnet/standard/clr" target="_blank" >摘自《微软官方文档/.Net 基础知识/执行模型/公共语言运行时(CLR)》</a>



<h3>元数据</h3>

> 除了生成IL, 面向CLR的每个编译器还要在每个托管模块中生成完整的**元数据(metadata)**。元数据简单地说就是一个数据表集合。一些数据表描述了模块中定义了什么(比如类型及其成员)，另一些描述了模块引用了什么(比如导入的类型及其成员)。元数据是一些老技术的超集。这些老技术包括COM的“类型库”(Type Library)和 “接口定义语言”(InterfaceDefinition Language, IDL)文件。但CLR元数据远比它们全面。另外，和类型库及IDL不同，元数据总是与包含IL代码的文件关联。事实上，元数据总是嵌入和代码相同的EXE/DLL文件中，这使两者密不可分。由于编译器同时生成元数据和代码，把它们绑定一起， 并嵌入最终生成的托管模块，所以**元数据和它描述的IL代码永远不会失去同步**。
>
> 元数据有多种用途，下面仅列举一部分。
>
>        1. 元数据避免了编译时对原生C/C++头和库文件的需求，因为在实现类型/成员的IL代码文件中，己包含有关引用类型/成员的全部信息。编译器直接从托管模块读取元数据。
>        2. Microsoft Visual Studio用元数据帮助你写代码。“ 智能感知”(IntelliSense)技术会解析元数据，告诉你一个类型提供了哪些方法、属性、事件和字段。对于方法，还能告诉你需要的参数。
>        3. CLR的代码验证过程使用元数据确保代码只执行“类型安全”的操作。(稍后就会讲到验证。)
>        4. 元数据允许将对象的字段序列化到内存块，将其发送给另一台机器，然后反序列化，在远程机器上重建对象状态。
>        5. 元数据允许垃圾回收器跟踪对象生存期。垃圾回收器能判断任何对象的类型，并从元数据知道那个对象中的哪些字段引用了其他对象。
>
> 摘自《CLR via C# 》第五页



<h3>FCL</h3>

> Framework类库(Framework Class Library, FCL)。FCL是一组程序集的统称。



<h3>CTS</h3>

> 通用类型系统(Common Type System, CTS)。
>
> 
>
> 通用类型系统定义了如何在公共语言运行时(CLR)中声明、使用和管理类型，同时也是运行时跨语言集成支持的一个重要组成部分。 常规类型系统执行以下功能：
>
> - 建立一个支持跨语言集成、类型安全和高性能代码执行的框架。
> - 提供一个支持完整实现多种编程语言的面向对象的模型。
> - 定义各语言必须遵守的规则，有助于确保用不同语言编写的对象能够交互作用。
> - 提供包含应用程序开发中使用的基元数据类型（如[Boolean](https://docs.microsoft.com/zh-cn/dotnet/api/system.boolean)、[Byte](https://docs.microsoft.com/zh-cn/dotnet/api/system.byte)、[Char](https://docs.microsoft.com/zh-cn/dotnet/api/system.char)、[Int32](https://docs.microsoft.com/zh-cn/dotnet/api/system.int32) 和 [UInt64](https://docs.microsoft.com/zh-cn/dotnet/api/system.uint64)）的库。
>
> <a href="https://docs.microsoft.com/zh-cn/dotnet/standard/base-types/common-type-system">摘自《微软官方文档/.Net 基础知识/执行模型/通用类型系统(CTS)》</a>



<h3>托管代码</h3>

> 使用 .NET 时，我们经常会遇到“托管代码”这个术语。 本文档解释这个术语的含义及其更多相关信息。
>
> 简而言之，**托管代码就是执行过程交由运行时管理的代码**。 在这种情况下，相关的运行时称为公共语言运行时 (CLR)，不管使用的是哪种实现（例如 [Mono](https://www.mono-project.com/)、.NET Framework 或 .NET Core/.NET 5+）。 CLR 负责提取托管代码、将其编译成机器代码，然后执行它。 除此之外，运行时还提供多个重要服务，例如自动内存管理、安全边界、类型安全，等等。
>
> 相反，如果运行 C/C++ 程序，则运行的代码也称为“非托管代码”。 在非托管环境中，程序员需要亲自负责处理相当多的事情。 实际的程序在本质上是操作系统 (OS) 载入内存，然后启动的二进制代码。 其他任何工作 - 从内存管理到安全考虑因素 - 对于程序员来说是一个不小的负担。
>
> 托管代码是使用可在 .NET 上运行的一种高级语言（例如 C#、Visual Basic、F# 等）编写的。 使用相应的编译器编译以这些语言编写的代码时，无法获得机器代码， 而是获得 **中间语言** 代码，然后运行时会对其进行编译并将其执行。 C++ 是这条规则的一个例外，因为它也能够生成可在 Windows 上运行的本机非托管二进制代码。
>
> <a href="https://docs.microsoft.com/zh-cn/dotnet/standard/managed-code">摘自《微软官方文档/.Net 基础知识/高级主题/什么是托管代码》</a>

