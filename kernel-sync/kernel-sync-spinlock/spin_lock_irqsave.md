# spin\_lock\_irqsave

## spin\_lock\_irqsave

`spin_lock_irq` 와 비슷하지만 인터럽트가 걸렸을 때 이전의 상태를 보존합니다.

> /include/linux/spinlock.h:381

```c
#define spin_lock_irqsave(lock, flags)				\
do {								\
	raw_spin_lock_irqsave(spinlock_check(lock), flags);	\
} while (0)
```

> /include/linux/spinlock.h:246

```c
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)

#define raw_spin_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		flags = _raw_spin_lock_irqsave(lock);	\
	} while (0)
...
#else

#define raw_spin_lock_irqsave(lock, flags)		\
	do {						\
		typecheck(unsigned long, flags);	\
		_raw_spin_lock_irqsave(lock, flags);	\
	} while (0)
...
#endif
```

SMP 환경인 경우 `flags` 에 `_raw_spin_lock_irqsave` 의 결과값을 저장하며, 이외에는 `_raw_spin_lock_irqsave` 에 `lock`, `flags` 2개의 인자를 전달합니다.



### \_raw\_spin\_lock\_irqsave: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:68

```c
#define _raw_spin_lock_irqsave(lock, flags)	__LOCK_IRQSAVE(lock, flags)
```

> /include/linux/spinlock\_api\_up.h:39

```c
#define __LOCK_IRQSAVE(lock, flags) \
  do { local_irq_save(flags); __LOCK(lock); } while (0)
```

`__LOCK_IRQSAVE` 매크로는 `local_irq_save` 매크로를 사용하고, `__LOCK` 으로 preemption을 비활성화합니다.

> /include/linux/irqflags.h:199

```c
#define local_irq_save(flags)					\
	do {							\
		raw_local_irq_save(flags);			\
	} while (0)
```

`local_irq_save` 은 `raw_local_irq_save` 을 실행합니다.

> /include/linux/irqflags.h:138

```c
#define raw_local_irq_save(flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		flags = arch_local_irq_save();		\
	} while (0)
```

`raw_local_irq_save` 는 `flags` 인자에 `arch_local_irq_save` 의 실행값을 저장하는 역할을 합니다.

> /include/x86/include/asm/lockspin.h:118

```c
static __always_inline unsigned long arch_local_irq_save(void)
{
	unsigned long flags = arch_local_save_flags();
	arch_local_irq_disable();
	return flags;
}
#else
```

`arch_local_irq_save` 함수는 `arch_local_save_flags` 를 호출해 결과값을 `flags` 변수에 저장하고, `arch_local_irq_disable` 을 호출해 인터럽트를 무시합니다.

> /arch/x86/include/asm/irqflags.h:77

```c
static __always_inline unsigned long arch_local_save_flags(void)
{
	return native_save_fl();
}
```

`arch_local_save_flags` 는 `native_save_fl` 함수를 사용합니다.

> /arch/x86/include/asm/irqflags.h:20

```c
extern __always_inline unsigned long native_save_fl(void)
{
	unsigned long flags;

	/*
	 * "=rm" is safe here, because "pop" adjusts the stack before
	 * it evaluates its effective address -- this is part of the
	 * documented behavior of the "pop" instruction.
	 */
	asm volatile("# __raw_save_flags\n\t"
		     "pushf ; pop %0"
		     : "=rm" (flags)
		     : /* no input */
		     : "memory");

	return flags;
}
```

`native_save_fl` 함수는 `pushf` 와 `pop` 명령어를 사용해 EFLAGS 레지스터 값을 `flags` 변수에 저장하고 해당 값을 리턴합니다.



### \_raw\_spin\_lock\_irqsave: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:58

```c
#ifdef CONFIG_INLINE_SPIN_LOCK_IRQSAVE
#define _raw_spin_lock_irqsave(lock) __raw_spin_lock_irqsave(lock)
#endif
```

> /kernel/locking/spinlock.c:156

```c
#ifndef CONFIG_INLINE_SPIN_LOCK_IRQSAVE
unsigned long __lockfunc _raw_spin_lock_irqsave(raw_spinlock_t *lock)
{
	return __raw_spin_lock_irqsave(lock);
}
EXPORT_SYMBOL(_raw_spin_lock_irqsave);
#endif
```

`CONFIG_INLINE_SPIN_LOCK_IRQSAVE` 가 정의되면 `__raw_spin_lock_irqsave` 를 매크로로 치환시키고, 그렇지 않을 경우 함수로 호출합니다.

> /include/linux/spinlock\_api\_smp.h:104

```c
static inline unsigned long __raw_spin_lock_irqsave(raw_spinlock_t *lock)
{
	unsigned long flags;

	local_irq_save(flags);
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	/*
	 * On lockdep we dont want the hand-coded irq-enable of
	 * do_raw_spin_lock_flags() code, because lockdep assumes
	 * that interrupts are not re-enabled during lock-acquire:
	 */
#ifdef CONFIG_LOCKDEP
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
#else
	do_raw_spin_lock_flags(lock, &flags);
#endif
	return flags;
}
```

`__raw_spin_lock_irqsave` 함수입니다. 먼저 `local_irq_save` 로 `flags` 에 EFLAGS 레지스터 값을 저장하고 `preempt_disable` 로 preemption을 비활성화합니다.

`CONFIG_LOCKDEP` 이 정의되어 있으면 `do_raw_spin_lock` 함수를 호출하고, 그렇지 않은 경우 `do_raw_spin_lock_flags` 함수를 호출합니다. 여기선 디버깅 관련 configuration을 무시하므로 `do_raw_spin_lock_flags` 를 살펴보겠습니다.

> /include/linux/spinlock.h:190

```c
static inline void
do_raw_spin_lock_flags(raw_spinlock_t *lock, unsigned long *flags) __acquires(lock)
{
	__acquire(lock);
	arch_spin_lock_flags(&lock->raw_lock, *flags);
	mmiowb_spin_lock();
}
```

`do_raw_spin_lock_flags` 는 `arch_spin_lock_flags` 를 호출합니다.

> /include/linux/spinlock.h:186

```c
#ifndef arch_spin_lock_flags
#define arch_spin_lock_flags(lock, flags)	arch_spin_lock(lock)
#endif
```

x86에서 `arch_spin_lock_flags` 는 `arch_spin_lock` 을 호출하도록 설정되어 있습니다.



## Conclusion

SMP: `spin_lock_irqsave` &gt; `_raw_spin_lock_irqsave` &gt; `__raw_spin_lock_irqsave` &gt; `do_raw_spin_lock_flags` &gt; `arch_spin_lock_flags` &gt; `arch_spin_lock` &gt; `queued_spin_lock` &gt; ...

UP: `spin_lock_irq` &gt; `_raw_spin_lock_irq` &gt; `__LOCK_IRQSAVE` &gt; `__LOCK` 

* EFLAGS/RFLAGS 레지스터 값을 유지하기 위해, 현재의 FLAGS 레지스터 값이 리턴됩니다. 이 값은 이후 FLAGS를 복원할 때 사용됩니다.

