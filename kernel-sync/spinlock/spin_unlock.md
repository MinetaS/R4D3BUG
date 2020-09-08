# spin\_unlock

## spin\_unlock

> /include/linux/spinlock.h:391

```c
static __always_inline void spin_unlock(spinlock_t *lock)
{
	raw_spin_unlock(&lock->rlock);
}
```

> /include/linux/spinlock.h:283

```c
#define raw_spin_unlock(lock)		_raw_spin_unlock(lock)
```



### \_raw\_spin\_unlock: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:75

```c
#define _raw_spin_unlock(lock)			__UNLOCK(lock)
```

`_raw_spin_unlock` 매크로가 `__UNLOCK` 으로 치환됩니다.

> /include/linux/spinlock\_api\_up.h:42

```c
#define ___UNLOCK(lock) \
  do { __release(lock); (void)(lock); } while (0)

#define __UNLOCK(lock) \
  do { preempt_enable(); ___UNLOCK(lock); } while (0)
```

`__UNLOCK` 매크로는 이전에 비활성화해뒀던 preemption을 다시 활성화합니다.



### \_raw\_spin\_unlock: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:70

```c
#ifndef CONFIG_UNINLINE_SPIN_UNLOCK
#define _raw_spin_unlock(lock) __raw_spin_unlock(lock)
#endif
```

> /kernel/locking/spinlock.c:180

```c
#ifdef CONFIG_UNINLINE_SPIN_UNLOCK
void __lockfunc _raw_spin_unlock(raw_spinlock_t *lock)
{
	__raw_spin_unlock(lock);
}
EXPORT_SYMBOL(_raw_spin_unlock);
#endif
```

왜 inline이 아니라 `CONFIG_UNLINE_SPIN_UNLOCK` 을 사용했는지는 모르겠지만 넘어갑시다. 해당 config가 정의되어 있으면 함수를 사용하고, 정의되어 있지 않으면 매크로를 사용해 `__raw_spin_unlock` 으로 치환합니다.

> /include/linux/spinlock\_api\_smp.h:148

```c
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
	spin_release(&lock->dep_map, _RET_IP_);
	do_raw_spin_unlock(lock);
	preempt_enable();
}
```

`__raw_spin_unlock` 함수입니다. `spin_release` 는 `spin_acquire` 와 마찬가지로 디버그 환경에서는 아무 것도 하지 않는 코드 \(`do {...} while (0)`\) 로 치환됩니다. `do_raw_spin_unlock` 을 호출하고, `preempt_enable` 로 preemption을 활성화합니다.

> /include/linux/spinlock.h:208

```c
static inline void do_raw_spin_unlock(raw_spinlock_t *lock) __releases(lock)
{
	mmiowb_spin_unlock();
	arch_spin_unlock(&lock->raw_lock);
	__release(lock);
}
```

`mmiowb_spin_unlock` 을 호출하고, 이어서 `arch_spin_unlock` 을 호출합니다.

> /include/asm-generic/qspinlock.h:114

```c
#define arch_spin_unlock(l)		queued_spin_unlock(l)
```

x86에서 `arch_spin_unlock` 은 `queued_spin_unlock` 을 의미합니다. `queued_spin_unlock` 은 Queued Spinlock 챕터에서 설명하도록 하겠습니다.



## Conclusion

SMP: `spin_unlock` &gt; `_raw_spin_unlock` &gt; `__raw_spin_unlock` &gt; `do_raw_spin_unlock` &gt; `arch_spin_unlock` &gt; `queued_spin_unlock` &gt; ...

UP: `spin_unlock` &gt; `_raw_spin_unlock` &gt; `__UNLOCK` 

* 프로세서가 1개인 경우 `spin_unlock`은 preemption을 활성화하는 것 외에 다른 동작을 하지는 않습니다.
* 다중 프로세서 환경인 경우 `spin_unlock` 은 `queued_spin_unlock` 을 사용합니다.

