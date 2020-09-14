# spin\_unlock\_irqrestore

## spin\_unlock\_irqrestore

lock을 release 하면서 플래그 레지스터를 복원합니다.

> /include/linux/spinlock.h:406

```c
static __always_inline void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags)
{
        raw_spin_unlock_irqrestore(&lock->rlock, flags);
}
```

> /include/linux/spinlock.h:286

```c
#define raw_spin_unlock_irqrestore(lock, flags)         \
        do {                                                    \
                typecheck(unsigned long, flags);                \
                _raw_spin_unlock_irqrestore(lock, flags);       \
        } while (0)
```

`raw_spin_unlock_irqrestore` 는 `_raw_spin_unlock_irqrestore` 를 호출합니다.



### \_raw\_spin\_lock\_irqsave: Uniprocessor \(UP\)

> /include/linux/spinlock\_api\_up.h:84

```c
#define _raw_spin_unlock_irqrestore(lock, flags) \
                                        __UNLOCK_IRQRESTORE(lock, flags)
```

> /include/linux/spinlock\_api\_up.h:55

```c
#define __UNLOCK_IRQRESTORE(lock, flags) \
  do { local_irq_restore(flags); __UNLOCK(lock); } while (0)
```

`__LOCK_IRQRESTORE` 는 `local_irq_restore` 매크로를 사용하고, `__UNLOCK` 으로 preemption을 다시 활성화합니다.

> /include/linux/irqflags.h:203

```c
#define local_irq_restore(flags) do { raw_local_irq_restore(flags); } while (0)
```

`local_irq_restore` 는 `raw_local_irq_restore`  실행합니다.

> /include/linux/irqflags.h:143

```c
#define raw_local_irq_restore(flags)                    \
        do {                                            \
                typecheck(unsigned long, flags);        \
                arch_local_irq_restore(flags);          \
        } while (0)
```

`arch_local_irq_restore` 를 사용해 플래그 레지스터를 복원합니다.

> /include/x86/include/asm/irqflags.h:82

```c
static __always_inline void arch_local_irq_restore(unsigned long flags)
{
        native_restore_fl(flags);
}
```

> /arch/x86/include/asm/irqflags.h:39

```c
extern inline void native_restore_fl(unsigned long flags)
{
        asm volatile("push %0 ; popf"
                     : /* no output */
                     :"g" (flags)
                     :"memory", "cc");
}
```

`native_restore_fl` 함수는 `push` 명령어로 기존에 저장해 두었던 플래그 레지스터 값을 스택에 올리고, `popf` 명령어를 사용해 해당 값을 EFLAGS/RFLAGS 레지스터를 복원합니다.



### \_raw\_spin\_lock\_irqrestore: Symmetric Multiprocessor \(SMP\)

> /include/linux/spinlock\_api\_smp.h:82

```c
#ifdef CONFIG_INLINE_SPIN_UNLOCK_IRQRESTORE
#define _raw_spin_unlock_irqrestore(lock, flags) __raw_spin_unlock_irqrestore(lock, flags)
#endif
```

> /kernel/locking/spinlock.c:188

```c
#ifndef CONFIG_INLINE_SPIN_UNLOCK_IRQRESTORE
void __lockfunc _raw_spin_unlock_irqrestore(raw_spinlock_t *lock, unsigned long flags)
{
        __raw_spin_unlock_irqrestore(lock, flags);
}
EXPORT_SYMBOL(_raw_spin_unlock_irqrestore);
#endif
```

`CONFIG_INLINE_SPIN_LOCK_IRQRESTORE` 정의 여부에 따라 `__raw_spin_unlock_irqrestore` 를 호출하는 방식이 달라집니다.

> /include/linux/spinlock\_api\_smp.h:155

```c
static inline void __raw_spin_unlock_irqrestore(raw_spinlock_t *lock,
                                            unsigned long flags)
{
        spin_release(&lock->dep_map, _RET_IP_);
        do_raw_spin_unlock(lock);
        local_irq_restore(flags);
        preempt_enable();
}
```

`do_raw_spin_unlock` 으로 `lock` 을 release 하고, `local_irq_restore` 로 플래그 레지스터를 복원합니다. 마지막으로 `preempt_enable` 으로 preemption을 활성화합니다.



## Conclusion

SMP: `spin_unlock_irqrestore` &gt; `_raw_spin_unlock_irqrestore` &gt; `__raw_spin_unlock_irqrestore` &gt; `do_raw_spin_unlock` &gt; `arch_spin_unlock` &gt; `queued_spin_unlock` &gt; ...

UP: `spin_unlock_irqrestore` &gt; `_raw_spin_unlock_irqrestore` &gt; `__UNLOCK_IRQRESTORE` &gt; `__UNLOCK` 

* 미리 저장해 두었던 값을 사용해 플래그 레지스터를 복원합니다.

