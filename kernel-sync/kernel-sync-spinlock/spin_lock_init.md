# spin\_lock\_init

## spin\_lock\_init

> /include/linux/spinlock.h:331

```c
#ifdef CONFIG_DEBUG_SPINLOCK

# define spin_lock_init(lock)					\
do {								\
	static struct lock_class_key __key;			\
								\
	__raw_spin_lock_init(spinlock_check(lock),		\
			     #lock, &__key, LD_WAIT_CONFIG);	\
} while (0)

#else

# define spin_lock_init(_lock)			\
do {						\
	spinlock_check(_lock);			\
	*(_lock) = __SPIN_LOCK_UNLOCKED(_lock);	\
} while (0)

#endif
```

`spin_lock_init` 은 매크로로 정의되어 있습니다. 여기서는 디버깅 관련 루틴은 다루지 않기 때문에 하단의 부분을 살펴보겠습니다.

먼저 `spinlock_check` 으로 `_lock` 객체를 검사하고, `__SPIN_LOCK_UNLOCKED` 을 사용해 `_lock` 을 초기화합니다.

> /include/linux/spinlock.h:326

```c
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}
```

`spinlock_check` 함수는 `lock`의 `raw_spinlock_t` 객체인 `rlock`의 주소를 리턴합니다.

> /include/linux/spinlock\_types.h:85

```c
#define ___SPIN_LOCK_INITIALIZER(lockname)	\
	{					\
	.raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,	\
	SPIN_DEBUG_INIT(lockname)		\
	SPIN_DEP_MAP_INIT(lockname) }

#define __SPIN_LOCK_INITIALIZER(lockname) \
	{ { .rlock = ___SPIN_LOCK_INITIALIZER(lockname) } }

#define __SPIN_LOCK_UNLOCKED(lockname) \
	(spinlock_t) __SPIN_LOCK_INITIALIZER(lockname)
```

다음은 `__SPIN_LOCK_UNLOCKED` 매크로입니다. `__SPIN_LOCK_INITIALIZER` 매크로를 사용하고 `spinlock_t` 타입으로 변환합니다.

`__SPIN_LOCK_INITIALIZER` 매크로는 `rlock` 필드에 `___SPIN_LOCK_INITIALIZER` 매크로를 사용한 값을 넣으며, `___SPIN_LOCK_INITIALIZER` 매크로는 `raw_lock` 필드에 `__ARCH_SPIN_LOCK_UNLOCKED` 값을 저장해 `raw_spinlock_t` 구조체를 생성합니다.

`SPIN_DEBUG_INIT` 매크로는 커널에 `CONFIG_DEBUG_SPINLOCK` config가 들어가지 않으면 아무 기능을 하지 않으며, 마찬가지로 `SPIN_DEP_MAP_INIT` 매크로는 `CONFIG_DEBUG_LOCK_ALLOC` config가 설정되어 있지 않은 경우 공백으로 치환됩니다. 여기서는 debug configuration을 무시하므로 해당 매크로에 대해서는 다루지 않고 넘어가겠습니다.

> /include/asm-generic/qspinlock\_types.h:57

```c
#define	__ARCH_SPIN_LOCK_UNLOCKED	{ { .val = ATOMIC_INIT(0) } }
```

x86에서 `__ARCH_SPIN_LOCK_UNLOCKED` 값은 0을 나타냅니다.

