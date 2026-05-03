# async callback lifetime

## 한 문장 정의

async callback lifetime은 timer, workqueue, RCU callback, completion handler처럼 나중에 실행되는 함수가 참조하는 object를 그 실행 시점까지 안전하게 살려두는 규칙이다.

## 왜 중요한가

동기 함수는 호출이 끝나면 stack과 지역 상태를 바로 추적할 수 있다. 반면 비동기 callback은 등록 시점과 실행 시점이 다르다. 그 사이에 원래 object가 해제되거나 상태가 바뀔 수 있다.

callback lifetime을 볼 때 핵심은 “callback이 실행될 때 이 pointer가 아직 유효한가”다.

```text
object 생성
    -> callback에 object pointer 저장
    -> callback 예약
    -> 다른 thread가 object destroy 시작
    -> callback 실행 가능성 확인
    -> 취소/대기/reference 정리
    -> free
```

## callback이 생기는 경로

커널에서 나중에 실행되는 함수는 여러 형태로 나타난다.

- timer callback: 지정 시점 이후 실행된다.
- workqueue: worker thread가 나중에 함수를 실행한다.
- delayed work: timer와 workqueue가 결합된다.
- RCU callback: grace period 이후 실행된다.
- completion path: I/O나 request가 끝난 뒤 실행된다.
- interrupt bottom half: interrupt 이후 deferred context에서 실행된다.

형태는 달라도 질문은 같다. callback이 들고 있는 pointer의 lifetime은 무엇으로 보장되는가?

## reference로 보장하는 방식

callback을 예약할 때 object reference를 얻고, callback이 끝날 때 put하는 방식이 있다.

```text
queue_work()
    -> object get
    -> worker callback 실행
    -> object 사용
    -> object put
```

이 방식은 callback이 실제로 실행되는 동안 object가 살아 있게 만든다. 대신 예약 실패, 취소 성공, 중복 queueing 같은 경로에서 get/put 짝을 정확히 맞춰야 한다.

## cancel로 보장하는 방식

destroy path에서 callback이 더 이상 실행되지 않도록 취소하고, 이미 실행 중이면 끝날 때까지 기다리는 방식도 있다.

자주 만나는 API 의미는 다음과 같이 읽으면 된다.

- `del_timer()`: pending timer를 제거하지만 이미 실행 중인 callback 대기는 별도 확인이 필요하다.
- `del_timer_sync()`: timer callback이 실행 중이면 끝날 때까지 기다리는 형태로 쓰인다.
- `cancel_work_sync()`: work가 pending 또는 running이면 취소하거나 완료를 기다린다.
- `flush_work()`: 이미 queue된 work가 끝날 때까지 기다린다.

정확한 API 의미는 context와 object 상태에 따라 달라지므로, destroy path가 어떤 보장을 필요로 하는지 먼저 봐야 한다.

## running callback과 destroy race

가장 위험한 상황은 destroy path와 callback path가 동시에 진행되는 경우다.

```text
CPU 0: destroy object 시작
CPU 1: callback 실행 시작
CPU 0: object free
CPU 1: callback이 object field 접근
```

이를 막으려면 적어도 하나가 필요하다.

- callback이 reference를 가지고 실행된다.
- destroy path가 callback 완료를 기다린다.
- object state를 먼저 DEAD로 바꾸고 새 callback 예약을 막는다.
- unregister로 외부 lookup을 끊은 뒤 pending callback을 drain한다.
- RCU나 lock으로 callback의 pointer 접근 범위를 보호한다.

## self-rescheduling callback

timer나 work가 자기 자신을 다시 예약하는 경우가 있다. destroy path는 pending callback 하나만 취소해서 충분하지 않을 수 있다.

확인할 점은 다음과 같다.

- destroy flag를 callback이 확인하는가?
- callback이 destroy 시작 이후 다시 queue되지 않는가?
- cancel 이후에도 다른 CPU가 새 callback을 예약할 수 없는가?
- repeated timer가 종료 조건 없이 다시 등록되지 않는가?

## 코드에서 확인할 것

1. callback에 저장된 pointer는 어떤 object를 가리키는가?
2. callback 예약 시 reference를 얻는가?
3. callback이 실행되지 않는 취소 경로에서는 reference를 누가 내려놓는가?
4. destroy path가 pending callback과 running callback을 모두 고려하는가?
5. callback이 자기 자신을 다시 예약할 수 있는가?
6. unregister, cancel, drain, free 순서가 올바른가?

## 보안 관점

async callback lifetime 버그는 재현이 어렵지만 영향이 크다.

- callback after free: object free 이후 timer나 work가 실행된다.
- double completion: cancel path와 completion path가 같은 request를 두 번 끝낸다.
- reference leak: queue 실패 또는 cancel 성공 경로에서 put을 놓친다.
- stale state: callback이 destroy 중인 object를 정상 object로 처리한다.
- requeue race: destroy 중인데 callback이 자신을 다시 예약한다.
- wrong credential: worker callback이 원래 요청자의 credential 없이 실행된다.

## 다른 문서와의 연결

- [cleanup path](cleanup-path.md): cancel, flush, drain, free 순서
- [refcount](refcount.md): callback 실행 동안 object를 살려두는 방식
- [ownership](ownership.md): callback 등록 후 해제 책임이 어디에 있는지
- [5. Concurrency](../5-concurrency/README.md): worker, timer, lock, race
- [12. io_uring](../12-io-uring/README.md): completion과 cancel의 lifetime 문제

## 기억할 문장

async callback을 읽을 때 핵심은 “이 함수가 나중에 실행될 때, 저장된 pointer가 무엇으로 살아 있는가?”다.
