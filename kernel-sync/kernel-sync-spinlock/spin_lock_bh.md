# spin\_lock\_bh

## spin\_lock\_bh

> /include/linux/spinlock.h:356

```c
static __always_inline void spin_lock_bh(spinlock_t *lock)
{
	raw_spin_lock_bh(&lock->rlock);
}
```

`spin_lock_bh` 내부에서 `raw_spin_lock_bh` 매크로를 사용합니다.

> /include/linux/spinlock.h:282

```c
#define raw_spin_lock_bh(lock)		_raw_spin_lock_bh(lock)
```

마찬가지로 `_raw_spin_lock_bh` 을 호출합니다.



### \_raw\_spin\_lock\_bh: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:62

```c
#define _raw_spin_lock_bh(lock)			__LOCK_BH(lock)
```

싱글 프로세서 환경이 `__LOCK_BH` 매크로를 사용합니다.

> /include/linux/spinlock\_api\_up.h:33

```c
#define __LOCK_BH(lock) \
  do { __local_bh_disable_ip(_THIS_IP_, SOFTIRQ_LOCK_OFFSET); ___LOCK(lock); } while (0)
```

`__local_bh_disable_ip` 함수를 호출합니다. 해당 함수는 `CONFIG_TRACE_IRQFLAGS` 의 설정 여부에 따라 동작이 달라집니다. `CONFIG_TRACE_IRQFLAGS` 은 디버그 빌드에서 설정되므로, 여기서는 해당 config가 false인 경우의 코드만을 살펴보도록 하겠습니다.

> /include/linux/bottom\_half.h:10

```c
static __always_inline void __local_bh_disable_ip(unsigned long ip, unsigned int cnt)
{
	preempt_count_add(cnt);
	barrier();
}
```

이 코드는 `preempt_disable` 과 유사한 형태입니다. `__preempt_count` 를 얼마나 증가시키는 지가 다릅니다. `__LOCK_BH` 에서는 `cnt` 에 `SOFTIRQ_LOCK_OFFSET` 값을 전달합니다. 이 값이 무엇을 의미하는지 확인하도록 하겠습니다.

> /include/linux/preempt.h:115

```c
/*
 * The preempt_count offset after spin_lock()
 */
#define PREEMPT_LOCK_OFFSET	PREEMPT_DISABLE_OFFSET

/*
 * The preempt_count offset needed for things like:
 *
 *  spin_lock_bh()
 *
 * Which need to disable both preemption (CONFIG_PREEMPT_COUNT) and
 * softirqs, such that unlock sequences of:
 *
 *  spin_unlock();
 *  local_bh_enable();
 *
 * Work as expected.
 */
#define SOFTIRQ_LOCK_OFFSET (SOFTIRQ_DISABLE_OFFSET + PREEMPT_LOCK_OFFSET)
```

`PREEMPT_LOCK_OFFSET` 은 `PREEMPT_DISABLE_OFFSET` 입니다. `SOFTIRQ_LOCK_OFFSET` 은, `SOFTIRQ_DISABLE_OFFSET` 과 `PREEMPT_LOCK_OFFSET` 을 더한 값입니다. 이 값을 `__preempt_count` 에 더하게 되면 다른 task의 preemption 뿐 아니라, 같은 CPU 내의 SoftIRQ를 막을 수 있게 됩니다. \(bottom halves는 현재 커널에서 deprecated 된 상태로, SoftIRQ 로 넘어갔습니다.\)



### \_raw\_spin\_lock\_bh: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:50

```c
#ifdef CONFIG_INLINE_SPIN_LOCK_BH
#define _raw_spin_lock_bh(lock) __raw_spin_lock_bh(lock)
#endif
```

> /kernel/locking/spinlock.c:173

```c
void __lockfunc _raw_spin_lock_bh(raw_spinlock_t *lock)
{
	__raw_spin_lock_bh(lock);
}
EXPORT_SYMBOL(_raw_spin_lock_bh);
#endif
```

`CONFIG_INLINE_SPIN_LOCK_BH` 이 정의되어 있으면 `__raw_spin_lock_bh` 를 호출하기 위해 매크로를 사용합니다.

> /include/linux/spinlock\_api\_smp.h:132

```c
static inline void __raw_spin_lock_bh(raw_spinlock_t *lock)
{
	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

`__raw_spin_lock` 과 같은 기능을 하지만, `preempt_disable` 대신 `__local_bh_disable_ip` 를 사용해 SoftIRQ도 막는 것을 볼 수 있습니다.



## Conclusion

SMP: `spin_lock_bh` &gt; `_raw_spin_lock_bh` &gt; `__raw_spin_lock_bh` &gt; `do_raw_spin_lock` &gt; `arch_spin_lock` &gt; `queued_spin_lock` &gt; ...

UP: `spin_lock_bh` &gt; `_raw_spin_lock_bh` &gt; `__LOCK_BH` 

* `spin_lock` 과의 차이점은, `preempt_disable` 대신 `__local_bh_disable_ip` 를 사용한다는 점입니다.

