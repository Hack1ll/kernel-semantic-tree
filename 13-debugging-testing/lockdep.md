# lockdep

## 한 문장 정의

`lockdep`은 커널이 runtime에 관찰한 lock 획득 관계를 기록해 deadlock 가능성과 lock context 위반을 찾는 검증 도구다.

## 왜 중요한가

커널 lock bug는 항상 즉시 deadlock으로 드러나지 않는다. 특정 순서와 CPU timing이 맞아야 멈추는 경우가 많다. lockdep은 실제로 lock을 잡은 순서를 graph로 쌓아, 앞으로 deadlock이 가능한 순환 관계를 발견하면 report한다.

이 문서의 줄기는 이것이다.

```text
lockdep은 지금 deadlock이 났는지가 아니라, 관찰된 lock 획득 규칙으로 deadlock이 가능해졌는지를 본다.
```

따라서 lockdep report는 “이미 멈춘 상태”보다 빠른 신호일 수 있다.

## lock class

lockdep은 lock instance를 그대로만 보지 않고 lock class를 추적한다.

구분해야 할 개념은 다음이다.

- lock instance: 실제 lock object 하나
- lock class: 같은 초기화 위치나 class key를 공유하는 lock 종류
- dependency edge: A를 잡은 상태에서 B를 잡았다는 관계
- chain: 현재 task가 잡고 있는 lock 순서

같은 type의 lock이라도 다른 class로 구분되어야 하는 경우가 있다. 반대로 서로 다른 object라도 같은 class로 묶어야 lock ordering을 제대로 볼 수 있다.

## lock order

deadlock의 핵심은 순서다.

```text
path 1: lock A -> lock B
path 2: lock B -> lock A
```

두 경로가 모두 가능하면 실제 deadlock이 아직 발생하지 않았더라도 위험하다. lockdep은 이런 cycle을 찾는다.

확인할 질문은 다음이다.

- report가 보여주는 이전 획득 순서와 현재 획득 순서는 무엇인가?
- 두 path가 실제로 동시에 실행될 수 있는가?
- 같은 lock class가 의도치 않게 공유되고 있지 않은가?
- nested locking이면 subclass annotation이 필요한가?

## context rule

lockdep은 lock 순서뿐 아니라 context 규칙도 본다.

대표 규칙은 다음이다.

- interrupt disabled 상태에서 잡을 수 있는 lock인가?
- hardirq context에서 잡는 lock인가?
- softirq context와 process context가 같은 lock을 공유하는가?
- sleep 가능한 context인가?
- mutex를 atomic context에서 잡지 않는가?

spinlock, mutex, rwlock은 허용 context가 다르다. lockdep report를 볼 때는 lock 종류와 실행 context를 함께 확인해야 한다.

## report 읽는 법

lockdep report에서 중요한 정보는 다음이다.

- 문제 유형
- 현재 task와 CPU
- 현재 lock chain
- 이전에 관찰된 반대 순서
- 각 lock의 class 이름
- lock을 획득한 stack
- interrupt context 관련 정보

읽는 순서는 다음이 좋다.

1. 어떤 lock 두 개가 문제인지 찾는다.
2. 현재 path의 lock order를 본다.
3. 이전 path의 lock order를 본다.
4. 두 path가 같은 object class를 공유하는지 확인한다.
5. 실제 설계상 허용되는 순서가 무엇인지 결정한다.
6. code path를 수정할지, annotation이 필요한지 판단한다.

## annotation

일부 lock pattern은 lockdep이 그대로 이해하기 어렵다. 이때 annotation이나 lock class 분리가 필요할 수 있다.

주의할 점은 다음이다.

- annotation은 실제 bug를 숨길 수 있다.
- class 분리는 설계상 독립 lock일 때만 맞다.
- nested locking은 실제 순서가 고정되어 있을 때만 안전하다.
- false positive라고 판단하기 전에 두 path가 동시에 가능한지 확인해야 한다.

lockdep을 조용하게 만드는 것이 목표가 아니다. 실제 lock 규칙을 명확히 만드는 것이 목표다.

## RCU와 lockdep

lockdep은 RCU read-side 관련 check와도 연결된다.

확인할 항목은 다음이다.

- RCU 보호 pointer를 RCU read-side 밖에서 dereference하지 않는가?
- `rcu_dereference`와 `rcu_assign_pointer` 규칙을 지키는가?
- sleep 가능한 RCU와 일반 RCU를 혼동하지 않는가?
- lock으로 보호된 pointer인지 RCU로 보호된 pointer인지 명확한가?

RCU 자체는 [5. Concurrency - RCU](../5-concurrency/rcu.md)에서 더 자세히 다룬다.

## 코드에서 확인할 것

lockdep report를 코드와 연결할 때는 다음을 찾는다.

- report에 나온 lock acquire stack
- lock initialization 위치
- 같은 lock class를 쓰는 다른 object
- lock을 잡기 전 이미 들고 있는 lock
- error path에서 unlock이 빠지는지
- callback, timer, worker에서 같은 lock을 잡는지
- interrupt enable/disable 상태

## 보안 관점

lockdep은 주로 안정성 도구지만, 보안 분석에도 중요하다.

- deadlock으로 denial-of-service 가능
- lock 누락이 race와 UAF로 이어짐
- atomic context 위반이 crash를 유발
- interrupt context lock misuse가 state corruption으로 이어짐
- false positive를 잘못 무시해 실제 race를 놓침

검토할 질문은 다음과 같다.

1. report가 가리키는 lock order cycle은 실제 실행 가능한가?
2. lock class가 너무 넓거나 너무 좁게 잡혀 있지 않은가?
3. 같은 객체 field를 보호하는 lock 규칙이 모든 path에서 같은가?
4. worker, timer, interrupt path에서도 같은 lock 순서를 지키는가?
5. annotation으로 report를 제거하기 전에 실제 설계가 확인되었는가?

## 다른 문서와의 연결

- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): KCSAN race report와 lockdep lock graph를 함께 볼 수 있다.
- [ftrace](ftrace.md): lock 획득 전후 함수 흐름을 추적할 수 있다.
- [5. Concurrency](../5-concurrency/README.md): lockdep은 concurrency 규칙을 runtime에 검증한다.
- [QEMU](qemu.md): CPU 수와 timing 설정은 lockdep report 재현에 영향을 준다.

## 기억할 문장

lockdep은 lock을 잘 잡았다는 확인 도구가 아니라, 관찰된 lock 순서가 미래의 deadlock 가능성을 만들었는지 검사하는 도구다.
