---
title: Linux Kernel - spinlock与睡眠
layout: post
guid: urn:uuid:b8199c34-1e63-4621-909f-211da861ad69
tags: tech
comments: no
---

   spinlock是linux内核锁机制的一种，而linux内核锁机制是linux内核同步机制的一部分。

   linux内核同步机制的使用目的是为了避免共享数据之间的竞争出现。它包括per cpu变量、原子操作、内存屏障、spinlock、信号量、顺序锁、禁止本地中断、禁止本地软中断、RCU等等。linux内核同步机制与SMP、抢占、可延迟函数、工作队列等等紧密关联。

   本文仅对于其中的一个问题进行分析，即：在linux内核代码中持有spinlock时为什么不能够睡眠。我们使用Linux内核2.6.11版本。

   首先，本质原因是spinlock的设计目的是保证数据修改的原子性，因此没有理由在spinlock锁住的区域内停留。

   然后我们来看具体实现上的原因。阅读内核源码之后发现，持有spinlock时为什么不能够睡眠的原因与SMP和内核抢占是否打开紧密相关。所以我们分以下四种情况来讨论：

## 单处理器不可抢占(!CONFIG\_SMP && !CONFIG\_PREEMPT)

  这种配置下，在内核中与spinlock相关的具体实现如下

```python
	#define _spin_lock(lock)        \
	do { \
		preempt_disable(); \空
		_raw_spin_lock(lock); \空
		__acquire(lock); \空
	} while(0)
	#define preempt_disable()               do { } while (0)
	#define _raw_spin_lock(lock)    do { (void)(lock); } while(0)
```

  即，实际上此时spin\_lock()是空操作。

  a. 在此种配置下正常使用spinlock的结果是：当前进程只能够被中断抢占（如果使用spin\_lock\_irq()甚至中断都不能够抢占），其他任何进程都不能换入，直到本进程完成临界区的执行。然而如果持有spinlock时睡眠则结果是：CPU会换入其他进程，可能对临界区数据进行修改，根本上破坏了使用锁的目的。

  b. 对于信号量来说，在此种配置下，即使进程持有信号量然后睡眠，且换入的其他进程想对临界区做修改，也会因为信号量的阻隔而不能成功。

## 单处理器可抢占(!CONFIG\_SMP && CONFIG\_PREEMPT)

  这种配置下，在内核中与spinlock相关的具体实现如下

```python
	#define _spin_lock(lock)        \
	do { \
		preempt_disable(); \
		_raw_spin_lock(lock); \空
		__acquire(lock); \空
	} while(0)
	#define preempt_disable() \
	do { \
		inc_preempt_count(); \
		barrier(); \
	} while (0)
```

  实际此时spin\_lock()上仅仅做了关闭抢占的操作，而且在关闭抢占之后就与第一种单处理器不可抢占的情况完全相同了。

## 多处理器不可抢占(CONFIG\_SMP && !CONFIG\_PREEMPT)

  这种配置下，在内核中与spinlock相关的具体实现如下

```python
	void __lockfunc _spin_lock(spinlock_t *lock)
	{
		preempt_disable();
		_raw_spin_lock(lock);
	}
	#define preempt_disable()               do { } while (0)
	static inline void _raw_spin_lock(spinlock_t *lock)
	{
	__asm__ __volatile__(
			spin_lock_string
			:"=m" (lock->slock) : : "memory");
	}
	#define spin_lock_string \
		"\n1:\t" \
		"lock ; decb %0\n\t" \
		"jns 3f\n" \
		"2:\t" \
		"rep;nop\n\t" \
		"cmpb $0,%0\n\t" \
		"jle 2b\n\t" \
		"jmp 1b\n" \
		"3:\n\t"
```

  spin\_lock()在这种配置下的操作实际上是做了对lock的原子减1，它的特点是在自旋等待获取lock时不可被抢占。

  a. 在此种配置下正常使用spinlock的结果是：当前进程在本处理器上只能被中断抢占，在其他处理器上若有进程想要访问该临界区，则由于试图获取被当前进程持有的lock而自旋等待，直到本进程执行完临界区，其他处理器上的等待进程才能进入临界区。如果当前进程在持有spinlock的时候睡眠：则本处理器上换入了其他进程，如果之后换入了一个想要获取同一个自旋锁执行同一段临界区的进程，则会停在自旋检测lock的值处，若多处理器上均换入了这样的进程，而原始的进程始终没得到机会执行并跳出临界区，则此时系统会死锁崩溃。

  b. 对于信号量来说：假设当前进程持有信号量然后睡眠，因为其他试图执行相同临界区的进程在获取信号量的时候不会自旋等待，这些进程会直接把自己放入等待队列然后调度自己，等待原始进程在执行完临界区之后做唤醒操作，所以不会产生死锁。

## 多处理器可抢占(CONFIG\_SMP && CONFIG\_PREEMPT)

```python
	void __lockfunc _##op##_lock(locktype##_t *lock) 
	{                                                                  
		preempt_disable();                                         
		for (;;) {                                                 
			if (likely(_raw_##op##_trylock(lock)))             
				break;                                     
			preempt_enable();                                  
			if (!(lock)->break_lock)                           
				(lock)->break_lock = 1;                    
			while (!op##_can_lock(lock) && (lock)->break_lock) 
				cpu_relax();                               
			preempt_disable();                                 
		}                                                          
	}                                                                  
	                                                                   
	EXPORT_SYMBOL(_##op##_lock);
	#define preempt_disable() \
	do { \
		inc_preempt_count(); \
		barrier(); \
	} while (0)
	static inline int _raw_spin_trylock(spinlock_t *lock)
	{
		char oldval;
		__asm__ __volatile__(
			"xchgb %b0,%1"
			:"=q" (oldval), "=m" (lock->slock)
			:"0" (0) : "memory");
		return oldval > 0;
	}
```

  spin\_lock()在这种配置下的操作实际上是做了preempt\_disable()和对lock的原子减1，特点是在自旋等待获取lock时可被抢占。

  与第3种配置不同的是此种配置下加入了自旋等待时可以被抢占的特性，本来的目的是减少自旋占用的系统时间。这个特性带来了一定改变：发生前述可能会导致死锁的情况时，由于可被抢占，系统仍然能够响应优先级高的进程，假如原始进程优先级很高，则可能解开死锁状态；如果没有解开，则自旋等待获取lock的进程在耗尽时间片后可能会被调入过期队列，然后原始进程可能会获得执行，解开死锁状态。但这些都只是可能，仍然有死锁的可能性存在。

  信号量的情况则与第3种相同。

总结：

  spinlock的具体实现与对称多处理器和内核抢占相关，在SMP和PREEMPT分别开启关闭一共四种的配置情况下，持有spinlock的时候睡眠均有可能产生灾难性的后果。即使碰巧不出现死锁或破坏临界区的情况，在持有spinlock的时候睡眠仍然是对系统资源的严重浪费，会导致系统性能严重下降，这也是持有spinlock的时候不可以睡眠的原因之一。
