# spin\_unlock\_bh

## spin\_unlock\_bh

> /include/linux/spinlock.h:396

```c
static __always_inline void spin_unlock_bh(spinlock_t *lock)
{
	raw_spin_unlock_bh(&lock->rlock);
}
```

> /include/linux/spinlock.h:291

```c
#define raw_spin_unlock_bh(lock)	_raw_spin_unlock_bh(lock)
```



### \_raw\_spin\_unlock\_bh: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:78

```c
#define _raw_spin_unlock_bh(lock)		__UNLOCK_BH(lock)
```

`_raw_spin_unlock_bh` 매크로가 `__UNLOCK_BH` 으로 치환됩니다.

> /include/linux/spinlock\_api\_up.h:48

```c
#define __UNLOCK_BH(lock) \
  do { __local_bh_enable_ip(_THIS_IP_, SOFTIRQ_LOCK_OFFSET); \
       ___UNLOCK(lock); } while (0)
```

`__UNLOCK_BH` 는 `__local_bh_enable_ip` 함수를 호출합니다.

> /kernel/softirq.c:166

```c
void __local_bh_enable_ip(unsigned long ip, unsigned int cnt)
{
	WARN_ON_ONCE(in_irq());
	lockdep_assert_irqs_enabled();
#ifdef CONFIG_TRACE_IRQFLAGS
	local_irq_disable();
#endif
	/*
	 * Are softirqs going to be turned on now:
	 */
	if (softirq_count() == SOFTIRQ_DISABLE_OFFSET)
		lockdep_softirqs_on(ip);
	/*
	 * Keep preemption disabled until we are done with
	 * softirq processing:
	 */
	preempt_count_sub(cnt - 1);

	if (unlikely(!in_interrupt() && local_softirq_pending())) {
		/*
		 * Run softirq if any pending. And do it in its own stack
		 * as we may be calling this deep in a task call stack already.
		 */
		do_softirq();
	}

	preempt_count_dec();
#ifdef CONFIG_TRACE_IRQFLAGS
	local_irq_enable();
#endif
	preempt_check_resched();
}
EXPORT_SYMBOL(__local_bh_enable_ip);
```

윗 부분은 디버깅에 관여하는 코드이므로 여기서는 무시하도록 하겠습니다.

`preempt_count_sub` 을 사용해 SoftIRQ를 다시 받도록 설정해 줍니다. 다음으로, 현재 인터럽트에 있지 않으며 pending 상태인 SoftIRQ가 존재한다면 `do_softirq` 함수를 호출해 밀려있는 IRQ를 처리해 줍니다. 마지막으로 `preempt_count_dec` 로 preemption을 비활성화합니다.



### \_raw\_spin\_unlock\_bh: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:74

```c
#ifdef CONFIG_INLINE_SPIN_UNLOCK_BH
#define _raw_spin_unlock_bh(lock) __raw_spin_unlock_bh(lock)
#endif
```

> /kernel/locking/spinlock.c:205

```c
#ifndef CONFIG_INLINE_SPIN_UNLOCK_BH
void __lockfunc _raw_spin_unlock_bh(raw_spinlock_t *lock)
{
	__raw_spin_unlock_bh(lock);
}
EXPORT_SYMBOL(_raw_spin_unlock_bh);
#endif
```

`CONFIG_INLINE_SPIN_UNLOCK_BH` 정의 여부에 따라, 매크로를 사용하거나 함수를 사용해 `__raw_spin_unlock_bh` 함수를 호출합니다.

> /include/linux/spinlock\_api\_smp.h:172

```c
static inline void __raw_spin_unlock_bh(raw_spinlock_t *lock)
{
	spin_release(&lock->dep_map, _RET_IP_);
	do_raw_spin_unlock(lock);
	__local_bh_enable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
}
```

`__raw_spin_unlock_bh` 함수입니다. `do_raw_spin_unlock` 을 호출한 뒤 `__local_bh_enable_ip` 로 SoftIRQ를 다시 받기 시작하면서 preemption도 활성화합니다.



## Conclusion

SMP: `spin_unlock_bh` &gt; `_raw_spin_unlock_bh` &gt; `__raw_spin_unlock_bh` &gt; `do_raw_spin_unlock` &gt; `arch_spin_unlock` &gt; `queued_spin_unlock` &gt; ...

UP: `spin_unlock_bh` &gt; `_raw_spin_unlock_bh` &gt; `__UNLOCK_BH` 

* lock이 해제되면서, pending 상태에 있던 SoftIRQ를 모두 처리합니다.

