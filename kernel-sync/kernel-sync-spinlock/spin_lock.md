# spin\_lock

## spin\_lock

> /include/linux/spinlock.h:351

```c
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

`spin_lock` 내부에서 `raw_spin_lock` 매크로를 사용하고, 인자로 `raw_spinlock_t` 객체를 넘겨줍니다.

> /include/linux/spinlock.h:224

```c
#define raw_spin_lock(lock)	_raw_spin_lock(lock)
```

`raw_spin_lock` 매크로는 `_raw_spin_lock` 함수를 호출하는 역할을 합니다.



### \_raw\_spin\_lock: Uniprocessor \(UP\)

`_raw_spin_lock`은 프로세서 개수에 따라서 매크로가 될 수도 있고 함수가 될 수도 있습니다. 프로세서가 1개, 즉 uniprocessor 인 경우 spinlock\_api\_up.h 안의 매크로들을 사용합니다.

> /include/linux/spinlock\_api\_up.h:58

```c
#define _raw_spin_lock(lock)			__LOCK(lock)
```

`_raw_spin_lock` 이 바로 `__LOCK` 매크로로 연결됩니다. 이 매크로 역시 같은 파일 안에 정의되어 있습니다.

> /include/linux/spinlock\_api\_up.h:27

```c
#define ___LOCK(lock) \
  do { __acquire(lock); (void)(lock); } while (0)

#define __LOCK(lock) \
  do { preempt_disable(); ___LOCK(lock); } while (0)
```

`__LOCK` 매크로는 preemption을 비활성화해 interrupt가 걸리는 것을 막고, 그 다음`___LOCK` 매크로를 사용합니다. `___LOCK` 매크로는 `__acquire` 매크로를 이용하며, 이 매크로는 컴파일 단계에서 lock/unlock 체크를 위한 context attribute를 넣는 역할을 하며 실제로 런타임에는 아무 영향을 끼치지 않습니다.

결론적으로, 프로세서의 개수가 1개인 경우 preemption을 enable 하는 것 외에 추가 동작을 하진 않습니다.



### \_raw\_spin\_lock: Symmetric Multiprocessor \(SMP\)

Symmetric multiprocessor \(SMP\) 환경인 경우 spinlock\_api\_smp.h 안의 함수를 사용합니다. 이 경우에는 `CONFIG_INLINE_SPIN_LOCK` config가 정의되어 있는지에 따라 구현이 달라집니다.

> /include/linux/spinlock\_api\_smp.h:46

```c
#ifdef CONFIG_INLINE_SPIN_LOCK
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
#endif
```

해당 config가 적용되었다면, `_raw_spin_lock` 을 매크로로 정의해 `__raw_spin_lock` 을 바로 호출합니다.

> /kernel/locking/spinlock.c:148

```c
#ifndef CONFIG_INLINE_SPIN_LOCK
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
EXPORT_SYMBOL(_raw_spin_lock);
#endif
```

해당 config가 적용되지 않은 경우, `_raw_spin_lock` 는 `__raw_spin_lock` 을 호출하는 함수의 이름을 나타냅니다.

> /include/linux/spinlock\_api\_smp.h:139

```c
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

`__raw_spin_lock` 함수는 `CONFIG_GENERIC_LOCKBREAK` 가 정의되지 않았거나 `CONFIG_DEBUG_LOCK_ALLOC` 가 정의되었을 때 선언 및 정의되는 함수입니다. Linux 커널에서`CONFIG_GENERIC_LOCKBREAK` 가 정의되는 아키텍쳐는 총 7개입니다. \(ia64, nds32, parisc, powerpc, s390, sh, sparc\)

UP 환경의 `__LOCK` 과 비슷하게, `preempt_disable` 매크로를 사용해 preemption을 비활성화합니다. `spin_acquire` 매크로는 최종적으로 `lock_acquire` 을 사용하며, `lock_acquire` 는 Kconfig 옵션에 따라 동작이 달라집니다. 먼저 아래의 2가지 옵션을 살펴보겠습니다.

> /lib/Kconfig.debug:1243

```text
config DEBUG_LOCK_ALLOC
	bool "Lock debugging: detect incorrect freeing of live locks"
	depends on DEBUG_KERNEL && LOCK_DEBUGGING_SUPPORT
	select DEBUG_SPINLOCK
	select DEBUG_MUTEXES
	select DEBUG_RT_MUTEXES if RT_MUTEXES
	select LOCKDEP
	help
	 This feature will check whether any held lock (spinlock, rwlock,
	 mutex or rwsem) is incorrectly freed by the kernel, via any of the
	 memory-freeing routines (kfree(), kmem_cache_free(), free_pages(),
	 vfree(), etc.), whether a live lock is incorrectly reinitialized via
	 spin_lock_init()/mutex_init()/etc., or whether there is any lock
	 held during task exit.
	
	config LOCKDEP
	bool
	depends on DEBUG_KERNEL && LOCK_DEBUGGING_SUPPORT
	select STACKTRACE
	select FRAME_POINTER if !MIPS && !PPC && !ARM && !S390 && !MICROBLAZE && !ARC && !X86
	select KALLSYMS
	select KALLSYMS_ALL
```

`CONFIG_DEBUG_LOCK_ALLOC` 과 `CONFIG_LOCKDEP` 는 모두 `DEBUG_KERNEL && LOCK_DEBUGGING_SUPPORT` 에 따라 값이 결정되기 때문에 항상 같은 값을 가집니다. 다음으로 아래의 코드를 살펴보겠습니다.

> /include/linux/lockdep.h:493

```c
# define lock_acquire(l, s, t, r, c, n, i)	do { } while (0)
# define lock_release(l, i)			do { } while (0)
# define lock_downgrade(l, i)			do { } while (0)
# define lock_set_class(l, n, k, s, i)		do { } while (0)
# define lock_set_subclass(l, s, i)		do { } while (0)
# define lockdep_init()				do { } while (0)
```

위의 매크는 `CONFIG_LOCKDEP` 가 false 일 때 정의됩니다. 따라서  `CONFIG_DEBUG_LOCK_ALLOC` 이 정의되지 않았다면 `spin_acquire` 매크로는 아무 동작도 하지 않는 빈 코드로 바뀌게 됩니다. 마찬가지로, 여기서는 디버깅 루틴을 살펴보지 않기 때문에 실제 `lock_acquire` 함수의 동작에 대해서는 설명을 건너뛰겠습니다.

`__raw_spin_lock` 의 마지막 줄에는 `LOCK_CONTENDED` 매크로가 사용되었습니다.

> /include/linux/lockdep.h:584

```c
#define LOCK_CONTENDED(_lock, try, lock)			\
do {								\
	if (!try(_lock)) {					\
		lock_contended(&(_lock)->dep_map, _RET_IP_);	\
		lock(_lock);					\
	}							\
	lock_acquired(&(_lock)->dep_map, _RET_IP_);			\
} while (0)
```

이 매크로는 `CONFIG_LOCK_STAT` 가 정의되었을 때 사용합니다. 이 config는 마찬가지로 디버깅 환경에서만 정의됩니다. 이외의 경우 `LOCK_CONTENDED` 는 아래와 같이 정의됩니다.

> /include/linux/lockdep.h:610

```c
#define LOCK_CONTENDED(_lock, try, lock) \
	lock(_lock)
```

`try` 인자를 사용하지 않고, `lock(_lock)` 을 호출합니다. 따라서 `do_raw_spin_trylock` 함수는 실제로 호출되지 않으며 `do_raw_spin_lock(lock)` 이 호출됩니다.

> /include/linux/spinlock.h:179

```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
	__acquire(lock);
	arch_spin_lock(&lock->raw_lock);
	mmiowb_spin_lock();
}
```

`arch_spin_lock` 을 호출해서 `qspinlock` lock을 걸고, `mmiowb_spin_lock` 을 호출합니다. `mmiowb_spin_lock` 내부 로직은 주제를 크게 벗어나는 범위이므로 여기서 다루지는 않겠습니다.

> /include/asm-generic/qspinlock.h:112

```c
#define arch_spin_lock(l)		queued_spin_lock(l)
```

`arch_spin_lock` 은 `queued_spin_lock` 을 사용합니다.

> /include/asm-generic/qspinlock.h:74

```c
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
	u32 val = 0;

	if (likely(atomic_try_cmpxchg_acquire(&lock->val, &val, _Q_LOCKED_VAL)))
		return;

	queued_spin_lock_slowpath(lock, val);
}
```

x86에서 `queued_spin_lock_slowpath` 의 정의는 따로 있습니다. 하지만 마찬가지로 내용이 범위를 벗어나는 만큼 여기서 자세히 다루지는 않고 코드만 올리도록 하겠습니다.

> /arch/x86/include/asm/qspinlock.h:48

```c
static inline void queued_spin_lock_slowpath(struct qspinlock *lock, u32 val)
{
	pv_queued_spin_lock_slowpath(lock, val);
}
```

> /arch/x86/include/asm/paravirt.h:653

```c
static __always_inline void pv_queued_spin_lock_slowpath(struct qspinlock *lock,
							u32 val)
{
	PVOP_VCALL2(lock.queued_spin_lock_slowpath, lock, val);
}
```

## Conclusion

SMP: `spin_lock` &gt; `_raw_spin_lock` &gt; `__raw_spin_lock` &gt; `do_raw_spin_lock` &gt; `arch_spin_lock` &gt; `queued_spin_lock` &gt; `queued_spin_lock_slowpath` &gt; `pv_queued_spin_lock_slowpath` &gt; PVOP\_VCALL

UP: `spin_lock` &gt; `_raw_spin_lock` &gt; `__LOCK` &gt; `___LOCK` 

