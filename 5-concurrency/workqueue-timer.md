# workqueue / timer

## 한 문장 정의

workqueue와 timer는 커널 작업을 현재 실행 문맥에서 바로 처리하지 않고 나중에 다른 문맥이나 지정된 시점에 실행하게 만드는 비동기 실행 장치다.

## 왜 중요한가

커널의 모든 작업을 지금 이 자리에서 처리할 수는 없다. interrupt handler는 오래 실행되면 안 되고, 어떤 작업은 sleep 가능한 process context가 필요하며, timeout 처리처럼 나중에 실행해야 하는 일도 있다.

workqueue와 timer는 실행 시점을 바꾸기 때문에 동시성 문제가 생긴다.

```text
현재 context
    -> work 또는 timer 등록
    -> 함수 return
    -> 나중에 callback 실행
    -> 원래 object 상태가 바뀌었을 수 있음
```

## workqueue

workqueue는 작업 함수를 worker thread에서 실행하게 한다. worker thread는 process context이므로 sleep 가능한 작업을 처리할 수 있다.

대표 흐름은 다음과 같다.

```text
INIT_WORK()
    -> queue_work()
    -> worker thread가 callback 실행
    -> callback 종료
```

workqueue는 interrupt나 atomic context에서 오래 할 수 없는 일을 나중으로 미루는 데 자주 쓰인다.

## timer

timer는 일정 시간이 지난 뒤 callback을 실행한다. timeout, retry, delayed cleanup, periodic check에 쓰인다.

대표 흐름은 다음과 같다.

```text
timer_setup()
    -> mod_timer()
    -> 시간이 지남
    -> timer callback 실행
```

timer callback은 sleep 가능한 일반 process context가 아니다. callback 안에서 어떤 lock과 helper를 사용할 수 있는지 별도로 확인해야 한다.

## delayed work

delayed work는 timer와 workqueue를 결합한 형태다. 일정 시간이 지난 뒤 worker thread에서 work callback을 실행한다.

```text
queue_delayed_work()
    -> timer 만료
    -> workqueue에 work 등록
    -> worker thread에서 실행
```

timer callback과 workqueue callback의 context 차이를 혼동하면 안 된다.

## object lifetime

work나 timer는 callback에 object pointer를 저장하는 경우가 많다. 등록한 뒤 callback이 실행되기 전에 object가 해제되면 UAF가 된다.

destroy path에서는 보통 다음 중 하나가 필요하다.

- callback 예약 시 reference를 얻고 callback 끝에서 put한다.
- destroy path에서 `cancel_work_sync()` 또는 timer sync API로 실행 가능성을 제거한다.
- callback이 다시 queue되지 않도록 state flag를 먼저 바꾼다.
- unregister로 외부 entry point를 닫은 뒤 pending work를 drain한다.

상세한 lifetime 문제는 [async callback lifetime](../4-object-lifetime/async-callback-lifetime.md)에서 이어진다.

## cancel, flush, drain

비동기 작업을 정리할 때 API 이름의 의미를 구분해야 한다.

- cancel: 아직 실행되지 않은 작업을 취소하려고 시도한다.
- sync cancel: 실행 중인 callback까지 고려해 끝날 때까지 기다릴 수 있다.
- flush: 이미 queue된 작업이 끝날 때까지 기다린다.
- drain: 새 작업이 추가되지 않도록 막고 남은 작업을 비운다.

정확한 보장은 API마다 다르므로, object free 전에 pending과 running 상태가 모두 제거되는지 봐야 한다.

## requeue와 periodic work

callback이 자기 자신을 다시 등록하는 구조가 있다.

```text
callback 실행
    -> 상태 확인
    -> 아직 필요하면 다시 queue
```

destroy path가 진행 중이면 callback이 다시 등록되지 않아야 한다. 보통 state flag, lock, atomic transition으로 이를 막는다.

## 코드에서 확인할 것

1. callback은 workqueue context인가, timer context인가?
2. callback 안에서 sleep 가능한 함수를 호출해도 되는가?
3. callback이 참조하는 object lifetime은 무엇으로 보장되는가?
4. destroy path가 pending callback과 running callback을 모두 처리하는가?
5. callback이 자기 자신을 다시 queue할 수 있는가?
6. cancel 또는 flush 이후 새 callback 등록을 막는 state rule이 있는가?

## 보안 관점

workqueue와 timer 버그는 비동기 race로 나타난다.

- object free 이후 work나 timer callback이 실행된다.
- cancel path와 callback path가 같은 request를 두 번 정리한다.
- timer context에서 sleep 가능한 API를 호출한다.
- worker가 원래 요청자의 credential 없이 작업을 수행한다.
- destroy 중인 object가 callback에서 다시 queue된다.
- flush는 했지만 새 queueing path를 막지 않아 작업이 다시 생긴다.

## 다른 문서와의 연결

- [mutex](mutex.md): worker context에서 sleep 가능한 lock을 쓰는 경우
- [spinlock](spinlock.md): timer나 interrupt와 공유하는 짧은 state 보호
- [atomic](atomic.md): cancel/completion state transition
- [async callback lifetime](../4-object-lifetime/async-callback-lifetime.md): callback pointer 수명
- [12. io_uring](../12-io-uring/README.md): worker와 completion/cancel path

## 기억할 문장

workqueue와 timer를 읽을 때 핵심은 “이 callback은 언제, 어떤 context에서, 어떤 object를 살아 있다고 믿고 실행되는가?”다.
