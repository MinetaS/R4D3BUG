# spin\_is\_locked

## spin\_is\_locked

> /include/linux/spinlock.h:444

```c
static __always_inline int spin_is_locked(spinlock_t *lock)
{
        return raw_spin_is_locked(&lock->rlock);
}
```

`raw_spinlock_t` 객체를 인자로 넘기면서 `raw_spin_is_locked` 를 호출합니다.

> /include/linux/spinlock.h:110

```c
#define raw_spin_is_locked(lock)        arch_spin_is_locked(&(lock)->raw_lock)
```

`raw_spin_is_locked` 는 `qspinlock` 을 기준으로, `arch_spin_is_locked` 를 사용합니다.

> /include/asm-generic/qspinlock.h:109

```c
#define arch_spin_is_locked(l)          queued_spin_is_locked(l)
```

`queued_spin_is_locked` 로 치환됩니다. 이 부분은 Queued Spinlock 챕터에서 이어서 설명됩니다.

