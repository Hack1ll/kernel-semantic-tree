# memory barrier

## 한 문장 정의

memory barrier는 CPU와 compiler가 memory access 순서를 바꾸는 것을 제한해, 다른 CPU가 값을 관찰하는 순서를 의도대로 맞추는 동기화 규칙이다.

## 왜 중요한가

커널 C 코드를 위에서 아래로 썼다고 해서 다른 CPU가 같은 순서로 값을 본다고 보장되지 않는다. compiler는 최적화를 위해 load/store를 바꿀 수 있고, CPU도 성능을 위해 memory operation을 재정렬할 수 있다.

lock을 쓰는 코드에서는 lock primitive가 ordering을 함께 제공하는 경우가 많다. 하지만 lockless code에서는 ordering을 직접 확인해야 한다.

```text
producer
    -> data 초기화
    -> ready flag publish

consumer
    -> ready flag 확인
    -> data 읽기
```

이 패턴에서 consumer가 ready를 본 뒤 초기화된 data를 보려면 release/acquire 같은 ordering이 필요하다.

## barrier가 해결하는 문제

memory barrier는 mutual exclusion을 제공하지 않는다. 여러 CPU가 동시에 같은 값을 쓰는 것을 막지 않는다.

barrier가 다루는 것은 순서다.

- store-store 순서
- load-load 순서
- load-store 순서
- store-load 순서
- compiler가 load/store를 합치거나 제거하는 문제

값의 소유권, object lifetime, critical section 보호는 다른 규칙으로 해결해야 한다.

## READ_ONCE와 WRITE_ONCE

`READ_ONCE()`와 `WRITE_ONCE()`는 compiler가 특정 load/store를 합치거나 반복하거나 재배치하는 것을 제한하는 데 쓰인다.

주로 다음 목적으로 사용된다.

- lockless flag 읽기와 쓰기
- data race가 의도된 상태 변수 접근
- compiler가 값을 여러 번 읽는 것을 막기
- pointer publish/read에서 명시적 단일 access 만들기

하지만 이것만으로 CPU 사이 ordering이 충분해지는 것은 아니다.

## acquire와 release

release는 이전 memory write가 publish 이후로 밀리지 않게 한다. acquire는 publish된 값을 읽은 뒤 이어지는 memory access가 그 앞으로 당겨지지 않게 한다.

```text
producer
    -> object field 초기화
    -> smp_store_release(&ready, 1)

consumer
    -> if (smp_load_acquire(&ready))
           object field 읽기
```

이 패턴은 ready flag를 본 consumer가 초기화된 object field를 읽도록 돕는다.

## full barrier

`smp_mb()` 같은 full memory barrier는 더 강한 ordering을 제공한다. 하지만 강한 barrier는 비용이 있고, 필요 이상으로 쓰면 코드 의미도 흐려진다.

barrier를 볼 때는 “무엇과 무엇 사이의 순서를 보장하려는가”를 명확히 해야 한다.

```text
write A
    -> barrier
write B
```

이 barrier가 필요한 이유를 설명할 수 없다면 잘못된 위치에 있을 가능성이 있다.

## lock과 ordering

spinlock, mutex, RCU, atomic API는 각자 ordering 의미를 가질 수 있다. 따라서 barrier를 직접 넣기 전에 기존 primitive가 이미 필요한 ordering을 제공하는지 확인해야 한다.

반대로 lockless fast path에서는 다음을 확인해야 한다.

- writer가 data를 초기화한 뒤 pointer를 publish하는가?
- reader가 pointer를 본 뒤 초기화된 data를 볼 수 있는가?
- flag와 data 사이에 acquire/release 관계가 있는가?
- `READ_ONCE()`만으로 충분한가, CPU barrier가 필요한가?

## 코드에서 확인할 것

1. 이 코드는 lockless로 shared state를 읽거나 쓰는가?
2. ordering을 맞춰야 하는 data와 flag가 무엇인가?
3. writer 쪽에는 release 의미가 있는가?
4. reader 쪽에는 acquire 의미가 있는가?
5. `READ_ONCE()`와 `WRITE_ONCE()`가 필요한 plain access가 있는가?
6. barrier가 mutual exclusion이나 lifetime 보장으로 오해되고 있지 않은가?

## 보안 관점

ordering 버그는 특정 architecture나 timing에서만 드러날 수 있다.

- ready flag를 봤지만 data 초기화가 보이지 않는다.
- pointer publish 전에 object field 초기화가 관찰되지 않는다.
- lockless state check가 오래된 값을 읽어 double completion을 허용한다.
- compiler가 반복 load를 최적화해 polling loop가 깨진다.
- barrier를 넣었지만 실제 필요한 방향의 ordering이 아니다.

## 다른 문서와의 연결

- [atomic](atomic.md): atomic 연산과 ordering 의미
- [RCU](rcu.md): pointer publish와 dereference ordering
- [spinlock](spinlock.md): lock이 제공하는 ordering과 lockless path 비교
- [13. Debugging / Testing](../13-debugging-testing/README.md): KCSAN과 concurrency 관찰

## 기억할 문장

memory barrier를 읽을 때 핵심은 “어떤 CPU가 어떤 값을 어떤 순서로 보아야 하는가?”다.
