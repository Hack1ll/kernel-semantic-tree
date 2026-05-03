# spinlock

## 한 문장 정의

spinlock은 sleep할 수 없는 짧은 critical section에서 공유 상태를 보호하기 위해 CPU를 양보하지 않고 기다리는 lock이다.

## 왜 중요한가

커널에는 process context뿐 아니라 interrupt, softirq, timer 같은 문맥도 있다. 이런 문맥에서는 기다리는 동안 sleep할 수 없다. 그래도 같은 queue, list, flag, counter를 여러 CPU가 동시에 바꾸면 상태가 깨진다.

spinlock은 이 상황에서 짧은 구간을 보호한다.

```text
CPU 0
    -> spin_lock()
    -> shared state update
    -> spin_unlock()

CPU 1
    -> 같은 lock을 얻을 때까지 busy wait
```

## 언제 쓰는가

spinlock은 다음 조건에서 자주 쓰인다.

- critical section이 짧다.
- sleep하면 안 되는 context에서 실행될 수 있다.
- interrupt 또는 softirq와 같은 state를 공유한다.
- list, queue, state flag 같은 작은 공유 상태를 즉시 갱신해야 한다.

반대로 긴 작업, I/O, user memory copy, blocking allocation을 spinlock 안에 넣으면 안 된다.

## interrupt와 softirq

같은 state를 interrupt handler도 만진다면 단순 `spin_lock()`만으로 부족할 수 있다. 현재 CPU에서 interrupt가 들어와 같은 lock을 다시 잡으려 하면 deadlock이 생길 수 있기 때문이다.

대표적으로 구분할 형태는 다음과 같다.

- `spin_lock()`: 일반 spinlock 획득
- `spin_lock_irqsave()`: local interrupt 상태를 저장하고 비활성화한 뒤 lock 획득
- `spin_lock_bh()`: softirq bottom half를 막고 lock 획득

어떤 variant가 필요한지는 “같은 lock을 어떤 execution context가 잡는가”로 결정한다.

## spinlock 안에서 피해야 할 것

spinlock을 잡은 동안에는 sleep 가능성이 있는 일을 피해야 한다.

- `mutex_lock()`처럼 sleep할 수 있는 lock 획득
- `copy_from_user()`처럼 page fault로 잠들 수 있는 경로
- `kmalloc(..., GFP_KERNEL)`처럼 reclaim 중 sleep할 수 있는 allocation
- blocking I/O
- 긴 loop 또는 복잡한 parser
- callback 실행이나 user-controlled 대기

spinlock 구간은 짧고 예측 가능해야 한다.

## lock이 보호하는 대상

spinlock 이름만 보고는 무엇이 안전한지 알 수 없다. 코드에서 확인해야 할 것은 lock 자체가 아니라 protected state다.

```text
struct foo {
    spinlock_t lock;
    struct list_head list;   protected by lock
    int state;               protected by lock
    atomic_t refcnt;         protected by atomic rule
};
```

같은 구조체 안의 모든 field가 같은 spinlock으로 보호된다고 가정하면 안 된다.

## unlock 이후 lifetime

spinlock은 critical section 안의 동시 접근을 막는다. 하지만 unlock 이후 object가 계속 살아 있다는 보장까지 자동으로 주지는 않는다.

다음 흐름은 위험할 수 있다.

```text
spin_lock()
    -> list에서 object pointer 찾기
spin_unlock()
    -> pointer 사용
```

unlock 뒤에도 pointer를 사용하려면 refcount, RCU, owner 규칙 같은 별도 lifetime 보장이 필요하다.

## 코드에서 확인할 것

1. 이 spinlock이 정확히 어떤 field를 보호하는가?
2. read path와 write path가 같은 spinlock을 쓰는가?
3. interrupt 또는 softirq에서도 같은 lock을 잡는가?
4. spinlock 안에서 sleep 가능한 함수를 호출하지 않는가?
5. lock ordering이 다른 path와 충돌하지 않는가?
6. unlock 뒤 pointer를 계속 쓴다면 lifetime 보장이 있는가?

## 보안 관점

spinlock 주변의 문제는 race와 deadlock으로 이어진다.

- write path만 lock을 잡고 read path는 lock 없이 읽는다.
- interrupt context와 process context가 같은 lock을 다른 방식으로 잡는다.
- spinlock 안에서 sleep해 커널 경고나 deadlock이 생긴다.
- lock을 푼 뒤 reference 없는 object pointer를 사용한다.
- lock ordering이 뒤집혀 deadlock이 생긴다.

## 다른 문서와의 연결

- [mutex](mutex.md): sleep 가능한 context에서 쓰는 lock
- [atomic](atomic.md): 단일 값 갱신을 lock 없이 처리하는 방식
- [memory barrier](memory-barrier.md): lockless path의 ordering 문제
- [4. Object Lifetime](../4-object-lifetime/README.md): unlock 이후 object lifetime
- [13. Debugging / Testing](../13-debugging-testing/README.md): lockdep으로 lock 규칙 점검

## 기억할 문장

spinlock을 읽을 때 핵심은 “어떤 context들이 같은 field를 만지고, 그 field를 잡은 동안 절대 sleep하지 않는가?”다.
