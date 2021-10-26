

# lab3 用户环境

　在这个实验中，我们将实现操作系统的一些基本功能，来实现用户环境下的进程的正常运行。你将会加强JOS内核的功能，为它增添一些重要的数据结构，用来记录用户进程环境的一些信息；创建一个单一的用户环境，并且加载一个程序运行它。你也可以让JOS内核能够完成用户环境所作出的任何系统调用，以及处理用户环境产生的各种异常。

## partA 用户环境与异常处理
新包含的文件'inc/env.h'包含了用户环境的基本定义，内核使用Env数据结构，追踪每个用户环境，在这个实验中，你会初始化一个环境，但是你需要设计JOS内核支持多个用户环境，在lab4会使用这个特性允许一个用户环境fork其他用户环境。   
在'kern/env.c'，内核维护三个全局变量环境：
```
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```
一旦JOS启动，envs指针便指向了一个 Env 结构体链表，表示系统中所有的用户环境的env。在我们的设计中，JOS内核将支持同一时刻最多 NENV 个活跃的用户环境，尽管这个数字要比真实情况下任意给定时刻的活跃用户环境数要多很多。系统会为每一个活跃的用户环境在envs链表中维护一个 Env 结构体。  
同时JOS内核会将没有用到env结构的中，用env_free_list的链表进行管理,这种设计方便用户环境env分配与回收。  
内核会把curenv指针指向正在执行的Env结构体，没有用户环境时，curenv = NULL。  
###  环境状态
　　我们要看一下，Env结构体每一个字段的具体含义是什么，Env结构体定义在 inc/env.h 文件中

　　struct Env {

　　　　struct Trapframe env_tf;      //saved registers

　　　　struct Env * env_link;         //next free Env

　　　　envid_t env_id;　　            //Unique environment identifier

　　　　envid_t env_parent_id;        //envid of this env's parent

　　　　enum EnvType env_type;　　//Indicates special system environment

　　　　unsigned env_status;　　   //Status of the environment

　　　　uint32_t env_runs;         //Number of the times environment has run

　　　　pde_t *env_pgdir;　　　　//Kernel virtual address of page dir.

　　};　　

　　env_tf:

　　　　这个类型的结构体在inc/trap.h文件中被定义，里面存放着当用户环境暂停运行时，所有重要寄存器的值。内核也会在系统从用户态切换到内核态时保存这些值，这样的话用户环境可以在之后被恢复，继续执行。

　　env_link:

　　　　这个指针指向在env_free_list中，该结构体的后一个free的Env结构体。当然前提是这个结构体还没有被分配给任意一个用户环境时，该域才有用。

　　env_id:

　　　　这个值可以唯一的确定使用这个结构体的用户环境是什么。当这个用户环境终止，内核会把这个结构体分配给另外一个不同的环境，这个新的环境会有不同的env_id值。

　　env_parent_id:

　　　　创建这个用户环境的父用户环境的env_id

　　env_type:

　　　　用于区别出来某个特定的用户环境。对于大多数环境来说，它的值都是 ENV_TYPE_USER.

　　env_status:

　　　　这个变量存放以下可能的值

　　　　ENV_FREE: 代表这个结构体是不活跃的，应该在链表env_free_list中。

　　　　ENV_RUNNABLE: 代表这个结构体对应的用户环境已经就绪，等待被分配处理机。

　　　　ENV_RUNNING: 代表这个结构体对应的用户环境正在运行。

　　　　ENV_NOT_RUNNABLE: 代表这个结构体所代表的是一个活跃的用户环境，但是它不能被调度运行，因为它在等待其他环境传递给它的消息。

　　　　ENV_DYING: 代表这个结构体对应的是一个僵尸环境。一个僵尸环境在下一次陷入内核时会被释放回收。

　　env_pgdir:

　　　　这个变量存放着这个环境的页目录的虚拟地址
就像Unix中的进程一样，一个JOS环境中结合了“线程”和“地址空间”的概念。线程通常是由被保存的寄存器的值来定义的，而地址空间则是由env_pgdir所指向的页目录表还有页表来定义的。为了运行一个用户环境，内核必须设置合适的寄存器的值以及合适的地址空间


### 分配用户环境数组
　在lab 2，你在mem_init() 函数中分配了pages数组的地址空间，用于记录内核中所有的页的信息。现在你需要进一步去修改mem_init()函数，来分配一个Env结构体数组，叫做envs。

#### Exercise 1 
    修改一下mem_init()的代码，让它能够分配envs数组。这个数组是由NENV个Env结构体组成的。envs数组所在的这部分内存空间也应该是用户模式只读的。被映射到虚拟地址UENVS处。

### 创建与运行环境
现在你需要去编写'kern/env.c'文件来运行一个用户环境了。由于你现在没有文件系统，所以必须把内核设置成能够加载内核中的静态二进制程序映像文件。

Lab3 里面的 GNUmakefile 文件在obj/user/目录下面生成了一系列的二进制映像文件。如果你看一下 kern/Makefrag 文件，你会发现一些奇妙的地方，这些地方把二进制文件直接链接到内核可执行文件中，只要这些文件是.o文件。其中在链接器命令行中的-b binary 选项会使这些文件被当做二进制执行文件链接到内核之后

#### 练习二 
        env_init(): 初始化所有的在envs数组中的 Env结构体，并把它们加入到 env_free_list中。 还要调用 env_init_percpu，这个函数要配置段式内存管理系统，让它所管理的段，可能具有两种访问优先级其中的一种，一个是内核运行时的0优先级，以及用户运行时的3优先级。

　　　　env_setup_vm(): 为一个新的用户环境分配一个页目录表，并且初始化这个用户环境的地址空间中的和内核相关的部分。

　　　　region_alloc(): 为用户环境分配物理地址空间

　　　　load_icode(): 分析一个ELF文件，类似于boot loader做的那样，我们可以把它的内容加载到用户环境下。

　　　　env_create(): 利用env_alloc函数和load_icode函数，加载一个ELF文件到用户环境中

　　　　env_run(): 在用户模式下，开始运行一个用户环境。










出现的问题及其解决方法：https://qiita.com/kagurazakakotori/items/334ab87a6eeb76711936