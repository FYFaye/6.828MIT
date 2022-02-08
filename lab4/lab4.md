# lab4 抢占式多任务机制

## 介绍
在这个实验中，你将在多并行的用户环境下实现抢占式多任务机制。  
在part A中，你需要在JOS中加入对多个任务处理器的支持，实现轮询调度机制，加入基本的环境管理的系统调用（通过这些进行创建与销毁用户环境，并且分配内存）。  
在part B中，你将会实现unix风格的fork()， 这个函数运行一个用户环境创建一个自己的拷贝。   
在part C中，你将会加入进程间通信机制(IPC)，运行不同的用户环境显式的进行通信与同步，你还需要加入对硬件时钟中断的支持，和抢占机制。   
## 开始
按照惯例，又有了一些新文件
```
    kern/cpu.h	内核私有的对多处理器的支持
    kern/mpconfig.c 读取多处理器配置的代码
    kern/lapic.c	内核驱动本地AIPC单元的代码
    kern/mpentry.S	非启动CPU的汇编代码entry
    kern/spinlock.h	内核私有的自旋锁，包括内核锁（保存内核的原子性的锁变量）
    kern/spinlock.c	内核的自旋锁代码
    kern/sched.c	你需要补全的调度框架代码

```

## PART A 多处理器支持与协作多任务
在实验的这部分中，我们首先拓展jos使其运行在多处理器系统上，然后实现jos内核一些系统功能调用以支持用户级环境去创建新环境，并且你需要实现协同轮询调度机制，当当前用户环境让出CPU资源时（或者退出时），允许内核从一个用户环境切换至另一个用户环境，在part C中，你还将实现抢占多任务机制，运行内核在一个确定的时间重新获得CPU资源，尽管当前的环境不合作。  
### 多处理器支持
我们现在即将实现JOS的“symetric multiprocessing”(SMP)机制，在这个机制下，所有cpu对系统资源（包括IO总线，内存）有平等的权限。尽管所有CPU在SMP机制中功能一致，不过在系统启动的过程中，他们被分为两类： 
* 启动处理器（Bootstrap Processor, BSP），负责系统的启动；
* 应用处理器(Application Processor, AP)，在BSP启动操作系统后被激活；  
哪一个CPU是BSP是由BIOS决定的，知道目前位置，所有JOS代码通过BSP运行。   
在SMP系统中，每个CPU有一个对应的本地APIC（LAPIC）单元，这个单元负责通过系统传递中断。LAPIC给与之关联的CPU一个唯一标志符，在这个实验中，我们使用下列的LAPIC的基本功能：
* 读取LAPIC的标志符（APIC ID），我们的代码目前是哪个CPU正在运行，函数cpunum()。   
* 从启动处理器到应用处理器，发送STARTUP处理器间中断，进而启动其他CPU，lapic_startup()。    
* 在part C中，我们补全LAPIC的时分，去出发时钟中断，支持抢占多任务, apic_init()。   

处理器通过memory-mapped IO(MMIO)访问其LAPIC，在MMIO中，一部分物理内存硬连接到其他IO设备，在这个情况下，使用存储与读取指令可以直接访问外部设备的寄存器。你已经看到物理地址在0xA0000（我们用于写入VGA显示显示器buffer），LAPIC单元存在起点物理地址为0xfe000000（也就是4G - 32MB），这么高的地址是不能KERNBASE直接进行访问，JOS虚拟地址在MMIOBASE映射了4MB的空隙，我们可以将设备映射到这个位置，在后续的实验中，会引入更多的MMIO区域，你将会写一个简单的函数分配地址从其他不同区域或者设备内存分配到这个地址。


#### 练习1
```
实现kern/pmap.c中的mmio_map_region函数，阅读kern/lapic.c的lapic_init函数，了解该函数是如何被调用的。在测试mmio_map_region进行前，你还有完成下一个练习。
```

------
根据注释的提示，该函数就是将[pa, pa + size]的区域，映射到c从MMIOBASE开始的[base, base + size]的区域，并返回base地址
* 首先需要round up为4k的整数倍
* base + size 不应该超过MMIOLIM，超过应该panic
* 需要更新静态变量base，并返回未更新前的地址。
```

//
// Reserve size bytes in the MMIO region and map [pa,pa+size) at this
// location.  Return the base of the reserved region.  size does *not*
// have to be multiple of PGSIZE.
//
void *
mmio_map_region(physaddr_t pa, size_t size)
{
	// Where to start the next region.  Initially, this is the
	// beginning of the MMIO region.  Because this is static, its
	// value will be preserved between calls to mmio_map_region
	// (just like nextfree in boot_alloc).
	static uintptr_t base = MMIOBASE;

	// Reserve size bytes of virtual memory starting at base and
	// map physical pages [pa,pa+size) to virtual addresses
	// [base,base+size).  Since this is device memory and not
	// regular DRAM, you'll have to tell the CPU that it isn't
	// safe to cache access to this memory.  Luckily, the page
	// tables provide bits for this purpose; simply create the
	// mapping with PTE_PCD|PTE_PWT (cache-disable and
	// write-through) in addition to PTE_W.  (If you're interested
	// in more details on this, see section 10.5 of IA32 volume
	// 3A.)
	//
	// Be sure to round size up to a multiple of PGSIZE and to
	// handle if this reservation would overflow MMIOLIM (it's
	// okay to simply panic if this happens).
	//
	// Hint: The staff solution uses boot_map_region.
	//
	// Your code here:
	size = ROUNDUP(size, PGSIZE);
	if (size + base > MMIOLIM)
		panic("mmio_map_region overflow");
	boot_map_region(kern_pgdir, base, size, pa, PTE_PCD | PTE_PWT | PTE_W);
	uintptr_t ret = base;
	base += size;
	return (void *)ret;
	//panic("mmio_map_region not implemented");
}

```
### Application Processor BootStrap

在启动应用处理器之前，BSP首先应该收集多处理器系统的信息，比如，总CPU数量，他们的APIC ID和LAPIC单元在MMIO的地址，'kern/mpconfig.c'中的mp_init()函数通过位于BIOS区域的MP配置表检索这类信息。

'kern/init.c'的boot_aps函数引导AP的bootstrap进程，AP启动与实模式，与boot.s中的bootloader类似，boot_aps()拷贝AP入口代码，到BSP实模式下的内存中，与bootloader不同的是，我们对于AP入口代码存放的位置可以有一定控制权：在JOS中使用MPENTRY_PADDR(0x7000)作为入口地址的存放位置，但是实际上640KB下任何未使用的地址都是可以使用的。

接下来，boot_aps()函数一个接一个的激活AP，通过向对应的AP发送STARTUP的IPI信号（处理器间中断），使用AP的入口代码初始化CS：IP地址（也就是entry code的起始地址）依次激活APs。AP就运行mpentry.s中的代码，从实模式运行到保护模式，最后mpentry.s调用C语言代码mp_main()('kern/init.c')，boot_aps等待AP将更新cpu_status到CPU_STATRTED，再去唤醒下一个AP。

#### 练习2
阅读boot_aps()与mp_main()与'kern/mpentry.s'代码，确保你理解了控制流的转换，然后修改page_init()避免在MPENTRY_PADDR的page加入空闲的页链表，由此可以安全的拷贝核运行AP的bootstrap代码，你的代码应该可用过测试check_page_free_list()。

-------
* 代码阅读
首先boot_aps()是BSP进行调用,这个函数主要是将mpentry的代码拷贝至0x7000处，同时引导AP启动。此时BSP已经运行在实模式下，所以应该是0x7000的虚拟地址。并通过函数lapic_startup()引导AP在0x7000启动，并等待CPU修改其自身状态。AP启动后，执行mpentry.s的代码；

```
// Start the non-boot (AP) processors.
static void
boot_aps(void)
{
	extern unsigned char mpentry_start[], mpentry_end[];
	void *code;
	struct CpuInfo *c;

	// Write entry code to unused memory at MPENTRY_PADDR
	code = KADDR(MPENTRY_PADDR);
	memmove(code, mpentry_start, mpentry_end - mpentry_start);

	// Boot each AP one at a time
	for (c = cpus; c < cpus + ncpu; c++) {
		if (c == cpus + cpunum())  // We've started already.
			continue;

		// Tell mpentry.S what stack to use 
		mpentry_kstack = percpu_kstacks[c - cpus] + KSTKSIZE;
		// Start the CPU at mpentry_start
		lapic_startap(c->cpu_id, PADDR(code));
		// Wait for the CPU to finish some basic setup in mp_main()
		while(c->cpu_status != CPU_STARTED)
			;
	}
}
```
* 练习解答
在page_init()中，将PGNUM(0x7000)的page不放在free list中即可。
```

//
// Initialize page structure and memory free list.
// After this is done, NEVER use boot_alloc again.  ONLY use the page
// allocator functions below to allocate and deallocate physical
// memory via the page_free_list.
//
void
page_init(void)
{
	// LAB 4:
	// Change your code to mark the physical page at MPENTRY_PADDR
	// as in use

	// The example code here marks all physical pages as free.
	// However this is not truly the case.  What memory is free?
	//  1) Mark physical page 0 as in use.
	//     This way we preserve the real-mode IDT and BIOS structures
	//     in case we ever need them.  (Currently we don't, but...)
	//  2) The rest of base memory, [PGSIZE, npages_basemem * PGSIZE)
	//     is free.
	//  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must
	//     never be allocated.
	//  4) Then extended memory [EXTPHYSMEM, ...).
	//     Some of it is in use, some is free. Where is the kernel
	//     in physical memory?  Which pages are already in use for
	//     page tables and other data structures?
	//
	// Change the code to reflect this.
	// NB: DO NOT actually touch the physical memory corresponding to
	// free pages!
	size_t i;
	page_free_list = NULL;

	//num_alloc：在extmem区域已经被占用的页的个数
	int num_alloc = ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE;
	//num_iohole：在io hole区域占用的页数
	int num_iohole = 96;

	for(i=0; i<npages; i++)
	{
		if(i==0)
		{
			pages[i].pp_ref = 1;
		}    
		else if(i >= npages_basemem && i < npages_basemem + num_iohole + num_alloc)
		{
			pages[i].pp_ref = 1;
		}
		else if (i == PGNUM(MPENTRY_PADDR))
			pages[i].pp_ref = 1;
		else
		{
			pages[i].pp_ref = 0;
			pages[i].pp_link = page_free_list;
			page_free_list = &pages[i];
		}
	}
}

```
#### 问题
* 比较'kern/mpentry.S'和'boot/boot.S'，记住两个代码都是编译连接后加载到KERNBASE之上运行的，为什么mpentry.S需要一个多余的宏定义MPBOOTPHYS？换句话说，如果在'kern/mpentry'省略它会出现什么问题？

因为mpentry.s传入的地址均为BSP的虚拟地址，在KERNBASE之上，而此时AP运行在16bits的实模式中，不可能访问到那么高的地址，通过该宏定义做一个内存映射，保证实模式能够正确访问。



### Per—CPU State and Initialization
当面向多处理器系统编程时，最重要的是区分每个cpu的状态对每个CPU来说是私有的，全局的状态是整个系统共享的，'kern/cpu.h'定义了大部分每个cpu的状态。包括'struct CpuInfo'，'cpunum'总是返回调用该函数的cpu id，该id一般被作为数组的索引值，比如cpus，宏定义'thiscpu'是当前cpu的'struct CpuInfo'的简写：
下面每个cpu值得注意的：
* 每个cpu的内核栈
* 每个cpu的TSS和TSS描述器
* 每个cpu的当前环境指针
* 每个cpu的系统寄存器

#### 练习3
修改'kern/pmap.c'中的'mem_init_mp()'，让CPU内核栈映射到相应的虚拟内存， 在KSTACKTOP开始，每个栈之间还有KSTKGAP的间隔，你的代码应该通过'check_kern_pgdir()'函数。

---
```

// Modify mappings in kern_pgdir to support SMP
//   - Map the per-CPU stacks in the region [KSTACKTOP-PTSIZE, KSTACKTOP)
//
static void
mem_init_mp(void)
{
	// Map per-CPU stacks starting at KSTACKTOP, for up to 'NCPU' CPUs.
	//
	// For CPU i, use the physical memory that 'percpu_kstacks[i]' refers
	// to as its kernel stack. CPU i's kernel stack grows down from virtual
	// address kstacktop_i = KSTACKTOP - i * (KSTKSIZE + KSTKGAP), and is
	// divided into two pieces, just like the single stack you set up in
	// mem_init:
	//     * [kstacktop_i - KSTKSIZE, kstacktop_i)
	//          -- backed by physical memory
	//     * [kstacktop_i - (KSTKSIZE + KSTKGAP), kstacktop_i - KSTKSIZE)
	//          -- not backed; so if the kernel overflows its stack,
	//             it will fault rather than overwrite another CPU's stack.
	//             Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	//
	// LAB 4: Your code here:
	for (uint32_t i = 0; i < NCPU; i ++){
		boot_map_region(kern_pgdir, KSTACKTOP - i * (KSTKSIZE + KSTKGAP) - KSTKSIZE, 
		KSTKSIZE, PADDR(percpu_kstacks[i]), PTE_W);
	}
}

```

#### 练习4
在'kern/trap.c'在trap_init_percpu()初始化了BSP的TSS和TSS描述器，在lab3中他是有效的，但是在其他CPU上是不正确的，修改这部分代码，使其对所有CPU均有效（tips：你的新代码不应该使用全局变量ts）

```
void
trap_init_percpu(void)
{
	// The example code here sets up the Task State Segment (TSS) and
	// the TSS descriptor for CPU 0. But it is incorrect if we are
	// running on other CPUs because each CPU has its own kernel stack.
	// Fix the code so that it works for all CPUs.
	//
	// Hints:
	//   - The macro "thiscpu" always refers to the current CPU's
	//     struct CpuInfo;
	//   - The ID of the current CPU is given by cpunum() or
	//     thiscpu->cpu_id;
	//   - Use "thiscpu->cpu_ts" as the TSS for the current CPU,
	//     rather than the global "ts" variable;
	//   - Use gdt[(GD_TSS0 >> 3) + i] for CPU i's TSS descriptor;
	//   - You mapped the per-CPU kernel stacks in mem_init_mp()
	//   - Initialize cpu_ts.ts_iomb to prevent unauthorized environments
	//     from doing IO (0 is not the correct value!)
	//
	// ltr sets a 'busy' flag in the TSS selector, so if you
	// accidentally load the same TSS on more than one CPU, you'll
	// get a triple fault.  If you set up an individual CPU's TSS
	// wrong, you may not get a fault until you try to return from
	// user space on that CPU.
	//
	// LAB 4: Your code here:
	int idx = thiscpu->cpu_id;
	struct Taskstate cpuTs = thiscpu->cpu_ts;
	// Setup a TSS so that we get the right stack
	// when we trap to the kernel.
	cpuTs.ts_esp0 = (uintptr_t)percpu_kstacks[idx];
	cpuTs.ts_ss0 = GD_KD;
	cpuTs.ts_iomb = sizeof(struct Taskstate);

	// Initialize the TSS slot of the gdt.
	gdt[(GD_TSS0 >> 3) + idx] = SEG16(STS_T32A, (uint32_t) (&thiscpu->cpu_ts),
					sizeof(struct Taskstate) - 1, 0);
	gdt[(GD_TSS0 >> 3) + idx].sd_s = 0;

	// Load the TSS selector (like other segment selectors, the
	// bottom three bits are special; we leave them 0)
	ltr(GD_TSS0 + (idx << 3));

	// Load the IDT
	lidt(&idt_pd);
}

```
在完成上述练习后，你使用4个CPU运行JOS'make qemu CPUS=4'，你会看到输出如下所示:
```
...
Physical memory: 66556K available, base = 640K, extended = 65532K
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_installed_pgdir() succeeded!
SMP: CPU 0 found 4 CPU(s)
enabled interrupts: 1 2
SMP: CPU 1 starting
SMP: CPU 2 starting
SMP: CPU 3 starting

```
### locking
我们现在的代码，在AP启动后会一直处于忙等待的状态（又叫自旋  spin），在AP进一步运行其他程序前，我们首先需要解决的是多CPU的内存，最简单的方法实现一个内核锁，内核锁是一个全局的锁变量，当一个环境进入到内核态时，这个锁变量被持有，在这个环境退出内核态时，释放该变量。在这个模式下，用户态的程序可以进行并发，但是只有一个环境可以进入到内核态，在内核态被占用时，任意想要进入内核态的环境被强制要求等待。

'kern/spinlock.h'声明了内核锁变量'kernal_lock'，同时其提供了'lock_kernal'和'unlock_kernal'两个函数对其进行操作。你应该在下面四个地方将其应用:

* 在i386_init()函数中，取得锁变量，在BSP唤醒其他CPU
* 在mp_main()中， 在初始化AP后，获得锁变量，然后调用sched_yield()在该AP中启动环境。
* 在trap中，当用户态通过trap进入内核态时，获取锁变量，为了判断，trap发生在内核态还是用户态，需要将检查tf_cs的low bits
* 在env_run中，在其切换至用户态时，释放锁变量，不能太早或者太晚，否则你会有竞争或者死锁的风险。

#### 练习5
根据上述要求应用锁变量

此时你暂时无法验证你的获取与释放锁变量的位置是否正确，应该是在你完成调度机制之后才可以验证。

#### 问题
使用锁变量保证了只有一个CPU可以进入到内核态，为什么还是需要对不同的内核使用不用内核栈，描述一个尽管使用了内核锁后，共用内核栈会发生错误的情况

---

在不同CPU由于中断发生切换时，？


### Round-Robin Scheduling
你的下一个任务是修改JOS内核，使其可以在多任务环境下进行以时间轮转的形式进行切换，Round-robin调度机制如下所示：
* 函数'sched_yield'在'kern/sched.c'中，是负责选择一个新环境进行运行，他顺序搜索envs数组，从上一个运行的环境开始，找到第一个ENV_RUNABLE的环境，然后通过env_run()切换到这个环境。

* 'sched_yield()'不能让相同的环境运行在两个CPU中，他可以通环境的状态是否为'ENV_RUNNING'进行判断。


* 我们已经实现了一个新的系统调用'sys_yield()'，用户态可以通过这个系统调用，自愿放弃CPU资源，从而切换到其他运行环境。

#### 练习6
在'sched_yield()'实现round-robin调度机制(也就是轮询调度机制)，不要忘记了在系统调用中解析sys_yield()，保证在mp_main()中调用sched_yield()

修改'kern/init.c'创建3个或者更多的环境，都运行程序'user/yield.c'

运行'make qemu'，你应该看到环境来回切换，在其终止之前，共来回切换5次，和下面类似:

```
...
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
...
```

---

* sched_yield()

```

// Choose a user environment to run and run it.
void
sched_yield(void)
{
	struct Env *idle;

	// Implement simple round-robin scheduling.
	//
	// Search through 'envs' for an ENV_RUNNABLE environment in
	// circular fashion starting just after the env this CPU was
	// last running.  Switch to the first such environment found.
	//
	// If no envs are runnable, but the environment previously
	// running on this CPU is still ENV_RUNNING, it's okay to
	// choose that environment.
	//
	// Never choose an environment that's currently running on
	// another CPU (env_status == ENV_RUNNING). If there are
	// no runnable environments, simply drop through to the code
	// below to halt the cpu.

	// LAB 4: Your code here.
	int i, index = 0;
	if (curenv)
		index = ENVX(curenv->env_id);
	else
		index = 0;

	for(i = index; i != index + NENV ; i++) {
		if (envs[i%NENV].env_status == ENV_RUNNABLE)
			env_run(&envs[i%NENV]);
	}
	if(curenv && curenv->env_status == ENV_RUNNING) {
		env_run(curenv);
	}
	// sched_halt never returns
	sched_halt();
}

```

* syscall

```
case SYS_yield:
			sys_yield();
			break;
```

