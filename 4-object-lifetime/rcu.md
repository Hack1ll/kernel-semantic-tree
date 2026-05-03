# RCU

## 한 문장 정의

RCU는 reader가 pointer를 보는 동안 writer가 object memory 재사용을 늦추어, read-heavy 자료구조를 빠르게 읽게 하는 lifetime과 동시성 기법이다.

## 왜 중요한가

커널에는 routing table, task list, credential, file descriptor table처럼 읽기가 매우 많은 자료구조가 있다. 모든 reader가 매번 무거운 lock을 잡으면 비용이 크다.

RCU는 reader를 빠르게 유지하는 대신 writer가 제거와 해제를 분리하게 만든다.

```text
reader
    -> rcu_read_lock()
    -> rcu_dereference(pointer)
    -> object 읽기
    -> rcu_read_unlock()

writer
    -> object를 lookup 구조에서 제거
    -> grace period 대기
    -> 그 뒤 free
```

## RCU가 보장하는 것

RCU는 “reader가 보고 있는 동안 object memory를 바로 재사용하지 않는다”는 보장을 제공한다.

하지만 RCU가 모든 것을 보장하지는 않는다.

- object field의 논리적 일관성을 자동으로 보장하지 않는다.
- writer 사이의 경쟁을 자동으로 막지 않는다.
- reader가 pointer를 RCU 구간 밖으로 들고 나가도 안전하게 해주지 않는다.
- object를 수정해도 되는 권한을 주지 않는다.

따라서 RCU pointer를 읽은 뒤 오래 사용하려면 별도 reference를 얻어야 할 수 있다.

## publish와 dereference

RCU pointer는 올바른 방식으로 publish되고 읽혀야 한다.

- writer는 `rcu_assign_pointer()` 같은 API로 새 pointer를 공개한다.
- reader는 `rcu_dereference()` 계열로 pointer를 읽는다.
- reader는 `rcu_read_lock()`과 `rcu_read_unlock()` 사이에서 접근한다.
- writer는 `synchronize_rcu()` 또는 `call_rcu()`로 해제를 늦춘다.

이 API들은 compiler와 CPU memory ordering 문제까지 포함해 다룬다. 일반 pointer 대입과 일반 pointer 읽기로 바꾸면 의미가 달라질 수 있다.

## grace period

grace period는 이전에 시작한 RCU reader들이 모두 빠져나갔다고 볼 수 있는 시점까지의 대기 기간이다.

```text
old object 제거
    -> 새 reader는 더 이상 old object를 찾지 못함
    -> 기존 reader는 old object를 볼 수 있음
    -> grace period 종료
    -> old object free 가능
```

이 구조 덕분에 writer는 lookup 구조에서 object를 먼저 제거하고, memory free는 나중으로 미룰 수 있다.

## RCU와 refcount 조합

RCU lookup 뒤 object를 RCU 구간 밖에서도 사용해야 하면 refcount와 함께 쓰는 경우가 많다.

```text
rcu_read_lock()
    -> pointer lookup
    -> refcount_inc_not_zero() 성공 확인
rcu_read_unlock()
    -> owned reference로 사용
    -> put
```

pointer를 찾았다는 사실과 reference를 얻었다는 사실은 다르다. refcount 증가에 실패하면 object는 이미 해제 절차에 들어갔을 수 있다.

## call_rcu와 synchronize_rcu

writer가 free를 늦추는 방식은 크게 두 가지로 볼 수 있다.

- `synchronize_rcu()`: 현재 thread가 grace period가 끝날 때까지 기다린다.
- `call_rcu()`: grace period 이후 실행될 callback을 등록한다.

`synchronize_rcu()`는 기다릴 수 있는 context에서만 적절하다. lock을 잡은 상태나 sleep 불가능한 context에서 사용할 수 있는지 반드시 확인해야 한다.

## 코드에서 확인할 것

1. RCU로 보호되는 pointer가 어떤 field인가?
2. reader가 `rcu_read_lock()` 범위 안에서만 pointer를 쓰는가?
3. writer가 object를 lookup 구조에서 먼저 제거하는가?
4. free가 grace period 이후에 일어나는가?
5. RCU 구간 밖에서 사용하려면 refcount를 얻는가?
6. pointer publish와 read에 RCU API를 쓰는가?

## 보안 관점

RCU 실수는 보기 어려운 UAF와 race를 만든다.

- RCU reader 밖으로 raw pointer를 들고 나간다.
- writer가 `kfree()`를 grace period 전에 호출한다.
- `call_rcu()` callback이 object 내부 resource 정리 순서를 잘못 처리한다.
- RCU lookup 뒤 refcount 획득 실패를 무시한다.
- writer끼리 같은 object를 동시에 제거하거나 교체한다.
- memory ordering을 무시해 초기화되지 않은 field를 reader가 본다.

## 다른 문서와의 연결

- [refcount](refcount.md): RCU lookup 뒤 오래 쓰기 위한 reference 획득
- [cleanup path](cleanup-path.md): unregister와 delayed free 순서
- [5. Concurrency](../5-concurrency/README.md): lock, atomic, memory ordering과의 관계
- [2. Process / Task](../2-process-task/README.md): task lookup과 RCU
- [9. Networking](../9-networking/README.md): routing, netfilter, socket lookup 경로

## 기억할 문장

RCU는 pointer를 빠르게 읽게 해주지만, RCU 구간 밖에서 object를 계속 쓰려면 별도 lifetime 보장이 필요하다.
