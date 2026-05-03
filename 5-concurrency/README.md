# 5. Concurrency

## 핵심 질문

같은 kernel state를 여러 CPU, task, interrupt, worker가 동시에 만질 때 무엇이 질서를 보장하는가?

## 큰가지의 의미

커널은 한 줄로 실행되지 않는다. syscall path, interrupt handler, softirq, timer, workqueue, 다른 CPU의 task가 같은 object를 동시에 볼 수 있다.

이 장은 lock API를 외우는 장이 아니다. 어떤 field가 어떤 규칙 아래에서 읽히고 바뀌는지 찾는 장이다.

```text
shared state
    -> access paths
    -> execution context
    -> protection rule
    -> lifetime after unlock
```

## 하위 문서의 역할

- [spinlock](spinlock.md): sleep 없이 짧은 critical section을 보호하는 lock
- [mutex](mutex.md): sleep 가능한 context에서 긴 작업을 보호하는 lock
- [atomic](atomic.md): 단일 값을 lock 없이 원자적으로 갱신하는 연산
- [memory barrier](memory-barrier.md): CPU와 compiler의 memory ordering을 제한하는 규칙
- [RCU](rcu.md): read-heavy path에서 reader를 가볍게 유지하는 동시성 방식
- [workqueue / timer](workqueue-timer.md): 작업을 나중에 다른 context에서 실행하는 비동기 장치

## 이 장에서 특히 구분할 것

lock을 썼다는 사실보다 중요한 것은 lock이 보호하는 대상이다.

```text
이 lock은 어떤 field를 보호하는가?
read path와 write path가 같은 lock을 쓰는가?
이 context는 sleep 가능한가?
lock을 푼 뒤 object는 아직 살아 있는가?
RCU read section 밖으로 pointer를 들고 나가지 않는가?
```

## 대표 흐름

```text
shared list update
    -> spinlock 획득
    -> list pointer 변경
    -> unlock
```

```text
read-mostly lookup
    -> rcu_read_lock
    -> pointer dereference
    -> use inside read section
    -> rcu_read_unlock
```

## 다른 큰가지와의 연결

- [4. Object Lifetime](../4-object-lifetime/README.md): concurrency bug는 lifetime bug로 이어지기 쉽다.
- [9. Networking](../9-networking/README.md): packet path와 rule update path가 동시에 돈다.
- [12. io_uring](../12-io-uring/README.md): cancel과 completion이 같은 request를 두고 경쟁한다.
- [13. Debugging / Testing](../13-debugging-testing/README.md): KCSAN과 lockdep으로 일부 concurrency bug를 관찰한다.

## 보안 관점

concurrency bug는 “검사한 상태”와 “사용한 상태”를 갈라놓는다.

- TOCTOU
- race-induced UAF
- double completion
- state corruption
- deadlock
- lockless ordering bug

## 읽고 나서 확인할 것

1. 공유되는 상태는 무엇인가?
2. 그 상태를 읽는 path와 쓰는 path는 각각 무엇인가?
3. 어떤 lock, RCU, atomic, barrier가 보호 규칙인가?
4. 실행 context가 sleep 가능한지 확인했는가?
5. unlock 이후에도 object lifetime이 보장되는가?
