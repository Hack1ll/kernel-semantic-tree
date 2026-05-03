# workers / cancellation

## 한 문장 정의

`workers / cancellation`은 즉시 처리하기 어려운 io_uring request를 worker에서 실행하고, 진행 중인 request를 취소하려는 경로를 다룬다.

## 왜 중요한가

io_uring은 빠른 제출 인터페이스지만, 모든 I/O가 즉시 끝나는 것은 아니다. 어떤 작업은 blocking 가능성이 있고, 어떤 작업은 event를 기다려야 하며, 사용자는 이미 제출한 request를 취소할 수도 있다.

이 문서의 줄기는 이것이다.

```text
worker와 cancellation은 같은 request를 다른 실행 경로에서 동시에 만지게 만드는 지점이다.
```

따라서 핵심은 worker 자체가 아니라 “worker, cancel, timeout, completion 중 누가 request의 마지막 상태를 결정하는가”다.

## worker로 이동하는 이유

request가 즉시 실행되지 못하면 async worker로 넘어갈 수 있다.

대표 이유는 다음이다.

- file operation이 blocking될 수 있음
- submit context에서 오래 실행하면 안 됨
- retry가 필요함
- filesystem이나 socket 경로가 잠을 잘 수 있음
- user가 SQPOLL을 쓰더라도 실제 작업은 별도 context가 필요함

worker 경로로 넘어가면 request는 submit task의 현재 stack과 분리되어 실행된다.

## io-wq

io_uring은 worker 실행을 위해 `io-wq` 계열 구조를 사용한다. worker는 request를 받아 실제 issue 함수를 다시 호출하거나 blocking 가능한 작업을 수행한다.

확인할 항목은 다음이다.

- bounded worker와 unbounded worker 구분
- worker가 사용할 credentials
- worker가 참조하는 ring context
- request를 worker queue에 넣을 때 reference가 증가하는가
- worker가 완료 또는 재시도 후 request를 어떻게 정리하는가

worker는 단순 background thread가 아니다. request lifetime과 permission 의미를 함께 운반한다.

## credentials와 personality

request는 제출한 task의 credentials 또는 등록된 personality를 사용해 실행될 수 있다. worker가 kernel thread라고 해서 worker 자신의 기본 권한으로 작업하면 안 된다.

확인할 질문은 다음이다.

- request에 어떤 cred가 저장되는가?
- worker 실행 전 cred override가 필요한가?
- error path에서 cred가 원래대로 복구되는가?
- fixed file의 `f_cred`와 request cred가 충돌하지 않는가?
- cancellation request는 누구의 권한으로 target을 찾는가?

io_uring worker 관련 권한 문제는 file operation의 실제 수행 주체와 연결된다.

## cancellation 종류

취소는 하나의 형태만 있는 것이 아니다.

대표 경로는 다음과 같다.

- explicit async cancel request
- timeout request
- linked timeout
- ring exit 중 pending request cancel
- file 또는 task 관련 request cancel
- poll wait에 걸린 request cancel
- worker queue에 있지만 아직 실행되지 않은 request cancel

취소가 성공했다는 말도 여러 의미를 가질 수 있다. 아직 실행되지 않은 request를 제거했을 수도 있고, 이미 완료 중이라 취소하지 못했을 수도 있다.

## cancel과 completion의 경쟁

가장 중요한 race는 이것이다.

```text
CPU 0: worker가 request를 완료하려 함
CPU 1: cancel path가 같은 request를 취소하려 함
```

안전한 구조라면 둘 중 하나만 user-visible completion을 만든다.

확인할 규칙은 다음이다.

- request 상태 전환이 atomic하게 결정되는가?
- cancel path가 이미 완료 중인 request를 다시 완료하지 않는가?
- worker가 cancel된 request를 계속 실행하지 않는가?
- timeout completion과 target request completion 순서가 정의되어 있는가?
- CQE가 request 의미에 맞게 한 번 또는 정해진 횟수만 기록되는가?

## poll과 waitqueue

일부 request는 event가 올 때까지 waitqueue에 걸린다. 이 경우 cancel은 waitqueue entry 제거와도 연결된다.

확인할 항목은 다음이다.

- waitqueue에 등록된 entry의 lifetime
- event callback과 cancel path의 동시 실행
- poll 재시작 중 request reference
- file이 닫히거나 ring이 종료될 때 poll entry 정리
- multi-shot poll request의 종료 조건

poll request는 worker 실행과 다른 방식으로 asynchronous state를 가진다.

## timeout과 linked timeout

timeout request는 시간이 지나면 completion을 만들거나 target request 취소를 시도한다. linked timeout은 특정 request chain과 함께 동작한다.

구분할 지점은 다음이다.

- timeout request 자체의 completion
- target request의 cancellation
- target이 이미 완료된 경우 timeout 처리
- timeout이 먼저 끝난 경우 linked request chain 처리
- timer callback과 normal completion의 동기화

timer callback은 request lifetime을 복잡하게 만든다. request가 timer에서 참조되는 동안 free되면 안 된다.

## ring teardown

process가 ring fd를 닫거나 task가 종료될 때 pending request가 남아 있을 수 있다.

teardown에서 확인할 항목은 다음이다.

- worker queue에 있는 request drain
- poll wait에 걸린 request 제거
- timeout timer cancel
- registered buffer와 fixed file cleanup 순서
- 남은 CQE 또는 overflow completion 처리
- ring context reference가 모든 request보다 오래 살아 있는가

teardown은 정상 I/O path가 아니라도 모든 lifetime 규칙을 만족해야 한다.

## 코드에서 확인할 것

workers/cancellation 코드를 읽을 때는 다음을 찾는다.

- request를 worker queue에 넣는 위치
- worker가 request를 꺼내 issue하는 경로
- cancel request가 target을 찾는 방식
- request state flag를 바꾸는 atomic operation
- timeout timer callback과 linked timeout 처리
- poll wait 등록과 제거 경로
- ring exit에서 pending work를 drain하는 순서

## 보안 관점

worker와 cancellation 관련 취약점은 주로 동시 상태 전환에서 나온다.

- worker completion과 cancel completion의 double completion
- cancel 후 worker가 request를 계속 사용
- timeout callback이 free된 request를 참조
- waitqueue entry 제거 race
- worker credential 복구 누락
- ring teardown 중 pending request UAF
- cancel 대상 lookup이 잘못된 request를 찾는 문제
- multi-shot request 종료 처리 오류

검토할 질문은 다음과 같다.

1. request는 worker queue에 들어간 순간 어떤 reference로 보호되는가?
2. cancel과 complete가 동시에 실행될 때 단 하나의 최종 상태만 선택되는가?
3. timeout timer와 target request lifetime은 서로 안전하게 묶여 있는가?
4. worker는 올바른 credentials로 file operation을 실행하는가?
5. ring exit 중 남은 worker, poll, timeout state가 모두 정리되는가?

## 다른 문서와의 연결

- [request lifetime](request-lifetime.md): worker와 cancel은 request lifetime을 가장 복잡하게 만드는 경로다.
- [SQ / CQ](sq-cq.md): cancel 성공과 worker 완료는 CQE로 전달된다.
- [registered buffers](registered-buffers.md): 취소되어도 buffer reference cleanup은 빠지면 안 된다.
- [fixed files](fixed-files.md): worker에서 file reference와 credentials 기준을 함께 확인해야 한다.

## 기억할 문장

io_uring의 worker와 cancellation을 볼 때는 누가 request를 실행하는지가 아니라, 누가 request의 최종 완료 상태를 단 한 번 결정하는지를 따라가야 한다.
