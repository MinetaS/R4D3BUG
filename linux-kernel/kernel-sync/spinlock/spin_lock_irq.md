# spin\_lock\_irq

## spin\_lock\_irq

> /include/linux/spinlock.h:376

```c
static __always_inline void spin_lock_irq(spinlock_t *lock)
{
        raw_spin_lock_irq(&lock->rlock);
}
```

> /include/linux/spinlock.h:281

```c
#define raw_spin_lock_irq(lock)         _raw_spin_lock_irq(lock)
```



### \_raw\_spin\_lock\_irq: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:65

```c
#define _raw_spin_lock_irq(lock)                __LOCK_IRQ(lock)
```

> /include/linux/spinlock\_api\_up.h:36

```c
#define __LOCK_IRQ(lock) \
  do { local_irq_disable(); __LOCK(lock); } while (0)
```

`__LOCK_IRQ` 는 `local_irq_disable` 매크로를 사용하고, 이후 `__LOCK` 으로 preemption을 비활성화합니다.

> /include/linux/irqflags.h:198

```c
#define local_irq_disable()     do { raw_local_irq_disable(); } while (0)
```

`CONFIG_TRACE_IRQFLAGS` 가 정의되지 않았을 때 `raw_local_irq_disable` 매크로만을 사용합니다.

> /include/linux/irqflags.h:136

```c
#define raw_local_irq_disable()         arch_local_irq_disable()
```

`raw_local_irq_disable` 은 다시 `arch_local_irq_disable` 을 호출합니다.

> /arch/x86/include/asm/irqflags.h:87

```c
static __always_inline void arch_local_irq_disable(void)
{
        native_irq_disable();
}
```

x86에서, `arch_local_irq_disable` 은 `native_irq_disable` 함수를 호출합니다.

> /arch/x86/include/asm/irqflags.h:47

```c
static __always_inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

`cli` 명령어는 EFLAGS 레지스터의 IF\(Interrupt Flag\)를 지우는 역할을 합니다. 이 명령어가 실행되고 나면 프로세서가 외부의 인터럽트를 무시하게 됩니다.



### \_raw\_spin\_lock\_irq: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:54

```c
#ifdef CONFIG_INLINE_SPIN_LOCK_IRQ
#define _raw_spin_lock_irq(lock) __raw_spin_lock_irq(lock)
#endif
```

> /kernel/locking/spinlock.c:165

```c
#ifndef CONFIG_INLINE_SPIN_LOCK_IRQ
void __lockfunc _raw_spin_lock_irq(raw_spinlock_t *lock)
{
        __raw_spin_lock_irq(lock);
}
EXPORT_SYMBOL(_raw_spin_lock_irq);
#endif
```

`CONFIG_INLINE_SPIN_LOCK_IRQ` 가 정의되면 `__raw_spin_lock_irq` 를 매크로로 치환시키고, 그렇지 않을 경우 함수로 호출합니다.

> /include/linux/spinlock\_api\_smp.h:124

```c
static inline void __raw_spin_lock_irq(raw_spinlock_t *lock)
{
        local_irq_disable();
        preempt_disable();
        spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
        LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

`local_irq_disable` 로 외부 인터럽트를 무시하고 preemption을 비활성화한 뒤, `LOCK_CONTENDED` 매크로를 사용해 `do_raw_spin_lock` 함수를 호출합니다.



## Conclusion

SMP: `spin_lock_irq` &gt; `_raw_spin_lock_irq` &gt; `__raw_spin_lock_irq` &gt; `do_raw_spin_lock` &gt; `arch_spin_lock` &gt; `queued_spin_lock` &gt; ...

UP: `spin_lock_irq` &gt; `_raw_spin_lock_irq` &gt; `__LOCK_IRQ` &gt; `__LOCK` 

* IRQ를 비활성화하기 위해 `local_irq_disable` 을 호출하여 EFLAGS 레지스터의 IF를 지우고 lock을 acquire 합니다.

