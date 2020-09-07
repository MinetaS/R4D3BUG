# Spinlock

이 글은 Linux Kernel 5.8.7 \(latest\) 을 기준으로 작성되었습니다.

## Structure of Spinlock

spinlock은 Linux 커널에서 사용하는 저수준 동기화 기법입니다.



> /include/linux/spinlock\_types.h:71

```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t;
```

kernel config에 CONFIG\_DEBUG\_LOCK\_ALLOC이 있으면 `__padding`과 `dep_map` 변수가 추가로 들어갑니다. 해당 옵션이 없다고 가정하면, `spinlock_t` 는 `struct raw_spinlock` 와 같은 타입이 됩니다.



> /include/linux/spinlock\_types.h:20

```c
typedef struct raw_spinlock {
	arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```

`arch_spinlock_t` 타입은 아케틱쳐에 따라 달라지는 spinlock 구현체입니다. x86의 경우 `struct qspinlock` 이 이에 해당합니다.



> /include/asm-generic/qspinlock\_types.h:52

```c
typedef struct qspinlock {
	union {
		atomic_t val;

		/*
		 * By using the whole 2nd least significant byte for the
		 * pending bit, we can allow better optimization of the lock
		 * acquisition for the pending bit holder.
		 */
#ifdef __LITTLE_ENDIAN
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
#else
		struct {
			u16	tail;
			u16	locked_pending;
		};
		struct {
			u8	reserved[2];
			u8	pending;
			u8	locked;
		};
#endif
	};
} arch_spinlock_t;
```

Atomic value을 사용하거나, endianness에 따라 locked, pending, tail 필드를 사용할 수 있습니다.

## Spinlock-related Operations

아래 목록은 spinlock과 관련한 동작을 하는 함수 또는 매크로입니다. spinlock은 acquired, released 두 가지 상태를 가지고 있으며 특정 thread가 해당 spinlock을 acquire 한 경우, 다른 thread가 동일한 spinlock을 acquire 하기 위해서는 해당 spinlock이 release 될 때까지 기다려야 합니다. 

* `spin_lock_init` : 주어진 spinlock을 unlocked 상태로 초기화합니다.
* `spin_lock` : 
* `spin_lock_bh` : 
* `spin_lock_irq` :
* `spin_lock_irqsave` :
* `spin_unlock` :
* `spin_unlock_bh` :
* `spin_is_locked` :

### spin\_lock\_init

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



> /include/linux/spinlock.h :326

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



### spin\_lock

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

dㅁ

