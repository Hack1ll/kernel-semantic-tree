# atomic

## 한 문장 정의

atomic 연산은 여러 CPU가 동시에 접근해도 단일 값의 read-modify-write가 중간에 깨지지 않도록 보장하는 lockless primitive다.

## 왜 중요한가

작은 counter나 state bit 하나를 바꾸기 위해 항상 lock을 잡을 필요는 없다. atomic 연산은 단일 값을 빠르게 갱신할 수 있게 한다.

하지만 atomic은 “단일 값”에 대한 보장이다. 여러 field가 함께 일관되어야 하는 구조에서는 lock이나 다른 동기화 규칙이 필요하다.

```text
atomic counter
    -> CPU 0 increment
    -> CPU 1 decrement
    -> 값 갱신이 서로 섞여 깨지지 않음
```

## atomic이 보장하는 것

atomic 연산은 특정 변수에 대한 갱신이 원자적으로 일어나게 한다.

대표적으로 다음 상황에 쓰인다.

- reference-like counter
- event counter
- bit flag set/clear
- state transition compare-and-swap
- one-time initialization guard

다만 “값이 원자적으로 바뀐다”와 “주변 data까지 올바른 순서로 보인다”는 다른 말이다. ordering은 별도 의미를 확인해야 한다.

## 자주 보는 연산

자주 만나는 형태는 다음과 같다.

- `atomic_read()`: atomic 변수 읽기
- `atomic_set()`: atomic 변수 설정
- `atomic_inc()`, `atomic_dec()`: 증가와 감소
- `atomic_dec_and_test()`: 감소 후 0 여부 확인
- `cmpxchg()`: 예상 값일 때만 새 값으로 교체
- `test_and_set_bit()`: bit를 세우면서 이전 상태 확인
- `set_bit()`, `clear_bit()`: bit flag 변경

API마다 memory ordering 의미가 다를 수 있다. 이름이 atomic이라고 해서 모든 순서 문제가 해결되는 것은 아니다.

## state transition

atomic은 state machine의 전이를 보호하는 데 자주 쓰인다.

```text
INIT
    -> RUNNING
    -> CLOSING
    -> DEAD
```

`cmpxchg()`를 쓰면 “현재 state가 RUNNING일 때만 CLOSING으로 바꾼다” 같은 조건부 전이를 구현할 수 있다. 이는 cancel path와 completion path가 동시에 같은 request를 끝내는 것을 막는 데 쓰일 수 있다.

## atomic으로 부족한 경우

다음 경우에는 atomic만으로 부족하다.

- 여러 field를 함께 업데이트해야 한다.
- list pointer와 count가 함께 일관되어야 한다.
- object lifetime이 pointer 접근과 함께 보장되어야 한다.
- update 중간 상태를 reader가 보면 안 된다.
- sleep 가능한 긴 작업을 보호해야 한다.

이 경우 spinlock, mutex, RCU, refcount API를 따로 봐야 한다.

## refcount와 atomic_t 구분

reference count를 단순 `atomic_t`로 구현하면 overflow, underflow, resurrected object 같은 문제가 묻힐 수 있다. lifetime 목적이면 `refcount_t`, `kref`, subsystem 전용 get/put API가 더 명확한 경우가 많다.

atomic counter가 lifetime을 의미하는지, 단순 통계 값을 의미하는지 구분해야 한다.

## 코드에서 확인할 것

1. atomic으로 보호하는 값은 정확히 하나인가?
2. 그 값과 함께 일관되어야 하는 다른 field가 없는가?
3. compare-and-swap 실패 경로를 처리하는가?
4. memory ordering이 필요한 publish/consume 패턴인가?
5. counter overflow 또는 underflow 가능성이 있는가?
6. lifetime 의미라면 refcount 전용 API가 필요한가?

## 보안 관점

atomic을 잘못 쓰면 lockless race가 숨어든다.

- atomic flag만 보고 object lifetime도 보장된다고 착각한다.
- counter는 맞지만 관련 list가 보호되지 않는다.
- `cmpxchg()` 실패를 무시해 state transition이 중복 실행된다.
- ordering 없는 flag publish로 reader가 초기화 전 data를 본다.
- refcount를 `atomic_t`로 다루다 underflow나 overflow를 놓친다.

## 다른 문서와의 연결

- [memory barrier](memory-barrier.md): atomic 연산의 ordering 의미
- [spinlock](spinlock.md): 여러 field를 함께 보호해야 할 때
- [refcount](../4-object-lifetime/refcount.md): lifetime 목적의 counter
- [workqueue / timer](workqueue-timer.md): cancel/completion state를 atomic transition으로 보호하는 경우
- [12. io_uring](../12-io-uring/README.md): request state 경쟁

## 기억할 문장

atomic을 읽을 때 핵심은 “원자적으로 바꾸는 값 하나만으로 이 상태 전체를 설명할 수 있는가?”다.
