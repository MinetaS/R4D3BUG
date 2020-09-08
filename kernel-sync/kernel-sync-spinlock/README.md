# Spinlock

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



