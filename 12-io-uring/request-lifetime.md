# request lifetime

## 한 문장 정의

`request lifetime`은 하나의 `io_kiocb`가 생성되고, 준비되고, 실행되고, 완료되거나 취소되고, 마지막 reference에서 해제되는 규칙이다.

## 왜 중요한가

io_uring request는 submit syscall의 stack frame에 묶여 있지 않다. 제출이 끝난 뒤에도 worker, poll, timeout, linked request, task work, completion path에서 계속 살아 있을 수 있다.

이 문서의 줄기는 이것이다.

```text
io_uring request는 SQE의 복사본이 아니라,
완료 전까지 file, buffer, cred, link 상태를 붙잡는 커널 객체다.
```

따라서 request를 볼 때는 “어디서 만들어졌는가”보다 “누가 아직 참조하고 있는가”가 더 중요하다.

## 큰 흐름

대표적인 request 생명주기는 다음과 같다.

```text
SQE 읽기
    -> io_kiocb 할당
    -> opcode별 prep
    -> file/buffer/cred reference 획득
    -> issue 시도
    -> 즉시 완료 또는 async 경로 이동
    -> completion 기록
    -> cleanup
    -> request put
```

모든 request가 이 순서를 단순하게 지나가지는 않는다. 일부 request는 재시도되고, 일부는 poll에 걸리고, 일부는 linked chain 안에서 대기한다.

## prep 단계

prep 단계에서는 SQE field를 opcode별 내부 상태로 옮긴다.

확인할 항목은 다음이다.

- opcode가 허용되는가?
- SQE의 flags 조합이 유효한가?
- fd 또는 fixed file slot이 올바른가?
- user buffer, offset, length 값이 overflow 없이 해석되는가?
- linked request나 timeout 관계가 만들어지는가?
- credentials 또는 personality가 request에 저장되는가?

prep 단계에서 검증이 부족하면 worker나 completion 단계에서 이미 잘못된 request를 처리하게 된다.

## issue 단계

issue 단계는 request를 실제 I/O operation으로 넘기는 지점이다.

가능한 결과는 여러 가지다.

- 즉시 성공하고 completion으로 이동
- 즉시 실패하고 error completion 기록
- blocking 가능성이 있어 worker로 이동
- poll wait에 등록
- timeout 또는 linked request 대기
- partial progress 후 재시도

request가 어느 경로로 이동하느냐에 따라 붙잡아야 하는 reference가 달라진다.

## request가 붙잡는 것

request는 완료 전까지 여러 객체를 참조할 수 있다.

대표 객체는 다음이다.

- `struct file`: 일반 fd lookup 또는 fixed file slot에서 얻은 file
- buffer state: user iov, registered buffer, provided buffer
- credentials: issue 시 사용할 cred 또는 personality
- ring context: completion을 기록할 대상
- linked request: 앞뒤 request 관계
- timeout object: 특정 request를 취소하거나 제한하는 timer
- poll entry: event 발생 시 재시작하기 위한 wait state

각 객체마다 get/put 위치가 다르다. request cleanup에서 무엇을 해제해야 하는지 opcode별로 확인해야 한다.

## 완료 규칙

request 완료는 user-visible result를 CQE로 남기는 일이다. 완료와 해제는 같은 말이 아니다.

구분해야 할 단계는 다음이다.

```text
complete: 결과를 결정하고 CQE 기록
cleanup: request가 붙잡은 부가 상태 정리
put: reference count 감소
free: 마지막 reference에서 메모리 해제
```

double completion은 같은 request가 user에게 두 번 결과를 보낸다는 뜻이다. request UAF는 completion 또는 cleanup 이후에도 다른 경로가 request를 만진다는 뜻이다.

## linked request

io_uring은 request를 chain으로 묶을 수 있다. 앞 request의 성공/실패가 뒤 request 실행 여부를 결정할 수 있다.

확인할 질문은 다음이다.

- link head와 follower의 reference가 어떻게 유지되는가?
- 앞 request 실패 시 뒤 request가 어떤 error로 완료되는가?
- linked timeout이 target request를 취소할 때 completion과 경쟁하지 않는가?
- chain 중간에서 cancel되면 남은 request의 cleanup이 맞는가?

linked request는 lifetime과 completion 순서를 복잡하게 만든다.

## multi-shot request

일부 request는 한 번 제출되고 여러 CQE를 낼 수 있다. 예를 들어 poll, accept, recv 계열 일부 동작은 조건에 따라 multi-shot으로 동작한다.

여기서는 다음을 구분한다.

- request 자체는 계속 살아 있다.
- CQE는 여러 번 나갈 수 있다.
- 마지막 CQE인지 flags로 표시될 수 있다.
- cancel이나 error가 multi-shot request를 끝낼 수 있다.

multi-shot에서는 “CQE가 한 번만 기록된다”는 일반 규칙이 그대로 적용되지 않는다. 대신 “request의 의미에 맞는 completion 횟수와 종료 조건”을 확인해야 한다.

## cleanup과 error path

request 준비 중 실패하거나 issue 중 실패하면 일부 reference만 잡힌 상태일 수 있다.

확인할 항목은 다음이다.

- prep 실패 시 잡은 file을 내려놓는가?
- buffer import 실패 시 iov나 page reference가 남지 않는가?
- worker로 넘기기 전 실패와 넘긴 뒤 실패가 구분되는가?
- completion 없이 silently drop되는 request가 없는가?
- ring exit 중 pending request가 안전하게 종료되는가?

error path는 정상 완료 경로보다 reference 균형이 깨지기 쉽다.

## 코드에서 확인할 것

request lifetime 코드를 읽을 때는 다음을 따라간다.

- `io_kiocb` allocation 위치
- opcode별 prep 함수
- file과 buffer reference 획득 지점
- async worker로 넘기는 지점
- completion 함수와 cleanup 함수
- request reference get/put 위치
- linked request와 timeout이 request를 찾는 방식
- ring teardown에서 pending request를 처리하는 방식

## 보안 관점

request lifetime 관련 취약점은 보통 다음 형태다.

- completion 후 request 재사용
- cancel과 complete의 double completion
- cleanup path에서 file 또는 buffer put 누락
- error path에서 일부 상태만 정리
- linked request chain의 dangling pointer
- multi-shot 종료 조건 오류
- request가 ring context보다 오래 살아남는 문제

검토할 질문은 다음과 같다.

1. request는 어떤 경로에서 완료될 수 있는가?
2. 완료와 해제가 명확히 분리되어 있는가?
3. 모든 reference는 획득한 경로와 실패 경로에서 균형을 이루는가?
4. cancel, timeout, worker가 동시에 같은 request를 만져도 한쪽만 완료하는가?
5. linked 또는 multi-shot request의 특수 규칙이 일반 cleanup과 충돌하지 않는가?

## 다른 문서와의 연결

- [SQ / CQ](sq-cq.md): SQE는 request를 만들고, CQE는 request 완료를 알린다.
- [registered buffers](registered-buffers.md): request가 buffer reference를 어떻게 붙잡는지 확인한다.
- [fixed files](fixed-files.md): request가 `struct file`을 어디서 얻고 언제 내려놓는지 연결된다.
- [workers / cancellation](workers-cancellation.md): async execution과 cancel race가 request lifetime의 핵심이다.

## 기억할 문장

io_uring request를 읽을 때는 submit 시점이 아니라 마지막 completion, cleanup, put이 끝나는 시점까지 따라가야 한다.
