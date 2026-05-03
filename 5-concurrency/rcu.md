# RCU

## 한 문장 정의

RCU는 읽기 경로를 가볍게 유지하고, writer가 pointer 교체와 object 해제를 단계적으로 처리하게 하는 read-mostly 동시성 기법이다.

## 왜 중요한가

어떤 자료구조는 update보다 lookup이 훨씬 많다. packet routing, socket lookup, task lookup, credential lookup 같은 경로에서 모든 reader가 mutex를 잡으면 성능 비용이 크다.

RCU는 reader가 짧은 read-side section에서 pointer를 읽게 하고, writer는 제거와 해제를 조심스럽게 분리한다.

```text
reader fast path
    -> rcu_read_lock()
    -> pointer lookup
    -> read-side section 안에서 사용
    -> rcu_read_unlock()

writer update path
    -> update lock 획득
    -> 새 pointer publish 또는 기존 pointer 제거
    -> reader가 빠져나간 뒤 old object 정리
```

## concurrency 관점의 RCU

이 문서에서는 RCU를 lifetime보다 read/write path 분리 관점에서 본다. object 해제 지연 자체는 [4. Object Lifetime의 RCU](../4-object-lifetime/rcu.md)에서 더 자세히 다룬다.

concurrency 관점에서 중요한 질문은 다음이다.

- 어떤 path가 reader인가?
- 어떤 path가 writer인가?
- reader는 lock 없이 어떤 field까지만 읽는가?
- writer는 어떤 lock으로 서로를 serialize하는가?
- reader가 본 object를 수정하지 않는가?

RCU reader가 빠르다는 말은 writer 동시성까지 자동으로 해결된다는 뜻이 아니다.

## reader 규칙

RCU reader는 read-side critical section 안에서 pointer를 읽고 사용한다.

```text
rcu_read_lock()
    -> rcu_dereference()
    -> object field 읽기
rcu_read_unlock()
```

reader는 보통 sleep하면 안 되는 RCU flavor를 쓴다. 커널에는 여러 RCU variant가 있으므로 현재 context에서 어떤 RCU 규칙이 적용되는지 봐야 한다.

reader가 section 밖으로 pointer를 들고 나가려면 refcount 같은 별도 lifetime 보장이 필요하다.

## writer 규칙

writer는 보통 update lock을 따로 사용한다. RCU는 reader를 가볍게 만들지만, writer끼리의 경쟁은 다른 lock이나 state rule로 막아야 한다.

writer path에서 확인할 점은 다음과 같다.

- 새 object를 완전히 초기화한 뒤 publish하는가?
- old object를 lookup 구조에서 제거한 뒤 free를 늦추는가?
- 여러 writer가 같은 pointer를 동시에 교체하지 못하게 막는가?
- reader가 보는 field를 update 중간 상태로 노출하지 않는가?

## RCU와 fast path

RCU는 fast path와 slow path를 분리할 때 자주 쓰인다.

```text
packet fast path
    -> RCU reader
    -> rule pointer 읽기
    -> packet 평가

configuration slow path
    -> mutex 또는 spinlock
    -> rule 교체
    -> old rule delayed free
```

이 구조에서는 reader가 어떤 field를 읽는지, writer가 어떤 시점에 새 field를 publish하는지 정확히 맞아야 한다.

## RCU로 부족한 경우

다음에는 RCU만으로 충분하지 않다.

- reader가 object를 수정해야 한다.
- reader가 long sleep section 밖에서 pointer를 계속 써야 한다.
- object 내부 여러 field를 writer가 동시에 바꾼다.
- writer끼리 서로 serialize되지 않는다.
- publish 전 초기화 ordering이 없다.

이 경우 refcount, lock, memory barrier, sequence counter 같은 다른 규칙이 필요하다.

## 코드에서 확인할 것

1. reader path와 writer path가 명확히 분리되어 있는가?
2. reader가 RCU section 안에서만 pointer를 사용하는가?
3. writer끼리는 어떤 lock이나 state rule로 serialize되는가?
4. 새 object는 완전히 초기화된 뒤 publish되는가?
5. reader가 보는 field를 writer가 in-place로 바꾸지 않는가?
6. RCU section 밖에서 쓰는 pointer는 reference를 얻는가?

## 보안 관점

RCU 동시성 실수는 lockless fast path에서 취약점이 된다.

- reader가 RCU section 밖에서 stale pointer를 사용한다.
- writer가 old object를 lookup 구조에서 제거하기 전에 free한다.
- writer끼리 충돌해 pointer 교체 순서가 깨진다.
- reader가 update 중간의 partially initialized object를 본다.
- fast path는 RCU로 읽는데 slow path cleanup이 refcount를 맞추지 않는다.

## 다른 문서와의 연결

- [memory barrier](memory-barrier.md): RCU pointer publish와 ordering
- [mutex](mutex.md): writer slow path를 serialize하는 lock
- [spinlock](spinlock.md): 짧은 update path와 RCU 조합
- [4. Object Lifetime RCU](../4-object-lifetime/rcu.md): grace period와 delayed free
- [9. Networking](../9-networking/README.md): packet path와 rule update path

## 기억할 문장

RCU를 concurrency 관점에서 읽을 때 핵심은 “reader는 무엇을 lock 없이 보고, writer는 무엇으로 교체 순서를 통제하는가?”다.
