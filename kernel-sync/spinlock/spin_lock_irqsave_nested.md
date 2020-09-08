# spin\_lock\_irqsave\_nested

## spin\_lock\_irqsave\_nested

> /include/linux/spinlock.h:386

```c
#define spin_lock_irqsave_nested(lock, flags, subclass)                 \
do {                                                                    \
        raw_spin_lock_irqsave_nested(spinlock_check(lock), flags, subclass); \
} while (0)
```

`spin_lock_irqsave` 와 비슷하지만, `subclass` 라는 인자가 추가되었습니다.

> /include/linux/spinlock.h:246

```c
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
...
#ifdef CONFIG_DEBUG_LOCK_ALLOC
...
#else
#define raw_spin_lock_irqsave_nested(lock, flags, subclass)             \
        do {                                                            \
                typecheck(unsigned long, flags);                        \
                flags = _raw_spin_lock_irqsave(lock);                   \
        } while (0)
#endif

#else
...
#define raw_spin_lock_irqsave_nested(lock, flags, subclass)     \
        raw_spin_lock_irqsave(lock, flags)

#endif

```

`CONFIG_DEBUG_LOCK_ALLOC` 이 정의되어 있지 않으면 `raw_spin_lock_irqsave_nested` 는 `raw_spin_lock_irqsave` 로 연결됩니다.

