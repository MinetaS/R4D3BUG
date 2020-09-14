# spin\_unlock\_irq

## spin\_unlock\_irq

> /include/linux/spinlock.h:401

```c
static __always_inline void spin_unlock_irq(spinlock_t *lock)
{
        raw_spin_unlock_irq(&lock->rlock);
}
```

> /include/linux/spinlock.h:284

```c
#define raw_spin_unlock_irq(lock)       _raw_spin_unlock_irq(lock)
```



### \_raw\_spin\_unlock\_irq: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:81

```c
#define _raw_spin_unlock_irq(lock)              __UNLOCK_IRQ(lock)
```

`_raw_spin_unlock_irq` 은 `__UNLOCK_IRQ` 으로 표현됩니다.

> /include/linux/spinlock\_api\_up.h:52

```c
#define __UNLOCK_IRQ(lock) \
  do { local_irq_enable(); __UNLOCK(lock); } while (0)
```

`__UNLOCK_IRQ` 는 `local_irq_enable` 를 실행하고 `__UNLOCK` 으로 preemption을 활성화합니다.

> /include/linux/irqflags.h:197

```c
#define local_irq_enable()      do { raw_local_irq_enable(); } while (0)
```

`local_irq_enable` 은 `raw_local_irq_enable` 로 치환됩니다.

> /include/linux/irqflags.h:137

```c
#define raw_local_irq_enable()          arch_local_irq_enable()
```

`raw_local_irq_enable` 은 다시 `arch_local_irq_enable` 로 치환됩니다.

> /arch/x86/include/asm/irqflags.h:92

```c
static __always_inline void arch_local_irq_enable(void)
{
        native_irq_enable();
}
```

x86에서, `arch_local_irq_enable` 함수는 `native_irq_enable` 함수를 호출합니다.

> /arch/x86/include/asm/irqflags.h:52

```c
static __always_inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

`native_irq_enable` 함수는 `sti` 명령어를 사용해 EFLAGS 레지스터의 IF\(Interrupt Flag\)를 1로 지정해 외부 인터럽트를 다시 받기 시작합니다.



### \_raw\_spin\_unlock\_irq: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:79

```c
#ifdef CONFIG_INLINE_SPIN_UNLOCK_IRQ
#define _raw_spin_unlock_irq(lock) __raw_spin_unlock_irq(lock)
#endif
```

> /kernel/locking/spinlock.c:196

```c
#ifndef CONFIG_INLINE_SPIN_UNLOCK_IRQ
void __lockfunc _raw_spin_unlock_irq(raw_spinlock_t *lock)
{
        __raw_spin_unlock_irq(lock);
}
EXPORT_SYMBOL(_raw_spin_unlock_irq);
#endif
```

`CONFIG_INLINE_SPIN_UNLOCK_IRQ` 가 정의되어 있으면 매크로를 사용하고, 그렇지 않으면 함수를 사용해 `__raw_spin_unlock_irq` 를 호출합니다.

> /include/linux/spinlock\_api\_smp.h:164

```c
static inline void __raw_spin_unlock_irq(raw_spinlock_t *lock)
{
        spin_release(&lock->dep_map, _RET_IP_);
        do_raw_spin_unlock(lock);
        local_irq_enable();
        preempt_enable();
}
```

`do_raw_spin_unlock` 으로 `lock` 을 release 하고, `local_irq_enable` 로 IF 플래그를 set 한 뒤 `preempt_enable` 로 preemption을 활성화합니다.



## Conclusion

SMP: `spin_unlock_irq` &gt; `_raw_spin_unlock_irq` &gt; `__raw_spin_unlock_irq` &gt; `do_raw_spin_unlock` &gt; `arch_spin_unlock` &gt; `queued_spin_unlock` &gt; ...

UP: `spin_unlock_irq` &gt; `_raw_spin_unlock_irq` &gt; `__UNLOCK_IRQ` &gt; `__UNLOCK` 

* unlock 후 `local_irq_enable` 함수를 사용해 EFLAGS 레지스터의 IF를 1로 설정하고, 인터럽트를 다시 처리하기 시작합니다.

