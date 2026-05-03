# SQ / CQ

## 한 문장 정의

`SQ`와 `CQ`는 user space와 kernel이 io_uring request를 주고받기 위해 공유하는 submission/completion ring이다.

## 왜 중요한가

io_uring의 성능은 syscall마다 request 전체를 복사하지 않는 데서 나온다. user space는 미리 mmap된 ring에 SQE를 채우고, kernel은 완료 결과를 CQE로 기록한다.

이 문서의 줄기는 이것이다.

```text
SQ/CQ는 user와 kernel이 같은 memory layout을 두고 head/tail 규칙으로 통신하는 경계다.
```

따라서 SQ/CQ를 볼 때는 queue 자료구조 자체보다 “누가 어떤 필드를 쓰고, 누가 어떤 순서로 읽는가”를 먼저 봐야 한다.

## 기본 구성

io_uring을 만들면 user space는 ring 관련 메모리를 mmap한다.

주요 구성은 다음이다.

- SQ ring: 제출할 SQE index를 담는 ring
- SQE array: 실제 request description이 들어 있는 배열
- CQ ring: 완료된 request의 CQE가 들어가는 ring
- shared head/tail: user와 kernel이 진행 위치를 주고받는 값
- ring flags: wakeup 필요 여부, overflow 상태 등 ring 상태

SQE와 CQE는 서로 다른 역할을 가진다.

```text
SQE: user가 kernel에게 요청하는 작업 설명
CQE: kernel이 user에게 돌려주는 완료 결과
```

## 제출 흐름

대표 제출 경로는 다음과 같다.

```text
user가 SQE 작성
    -> SQ tail 증가
    -> io_uring_enter 호출 또는 SQPOLL wakeup
    -> kernel이 SQ head/tail 확인
    -> SQE를 io_kiocb request로 변환
    -> request issue
```

중요한 점은 SQE가 user memory에 있다는 것이다. kernel은 SQE 내용을 읽어 내부 request로 준비한다. 준비가 끝난 뒤에는 내부 request의 상태가 기준이 된다.

## 완료 흐름

완료는 CQE로 전달된다.

```text
request 완료
    -> result와 flags 결정
    -> CQE slot 확보
    -> CQE 기록
    -> CQ tail 증가
    -> user가 CQ head를 이동하며 수거
```

CQE에는 보통 다음 정보가 들어간다.

- `user_data`: SQE에서 user가 지정한 식별자
- `res`: 결과 값 또는 음수 errno
- `flags`: buffer selection, more completion 등 추가 상태

`user_data`는 kernel object pointer가 아니다. user가 완료를 구분하기 위해 넣은 값이다.

## head와 tail 규칙

ring은 head와 tail로 빈 공간과 사용 중인 공간을 구분한다.

확인할 규칙은 다음이다.

- producer는 entry를 먼저 쓰고 tail을 갱신해야 한다.
- consumer는 tail을 확인한 뒤 entry를 읽고 head를 갱신해야 한다.
- head/tail 계산에서 ring size mask를 올바르게 적용해야 한다.
- full/empty 판정이 overflow 없이 계산되어야 한다.
- user가 head/tail을 조작해도 kernel이 범위를 벗어나면 안 된다.

SQ에서는 user가 producer이고 kernel이 consumer다. CQ에서는 kernel이 producer이고 user가 consumer다.

## memory ordering

SQ/CQ는 user와 kernel이 동시에 보는 memory다. 단순한 load/store처럼 보여도 순서 보장이 필요하다.

핵심 규칙은 다음이다.

- SQE 내용을 모두 쓴 뒤 SQ tail이 보이게 해야 한다.
- kernel은 tail을 본 뒤 SQE 내용을 읽어야 한다.
- kernel은 CQE 내용을 모두 쓴 뒤 CQ tail을 갱신해야 한다.
- user는 CQ tail을 본 뒤 CQE 내용을 읽어야 한다.

이 순서가 깨지면 kernel이 덜 작성된 SQE를 읽거나, user가 덜 작성된 CQE를 읽을 수 있다.

## SQPOLL과 IOPOLL

SQPOLL은 kernel thread가 submission queue를 polling하는 방식이다. user가 매번 `io_uring_enter`를 호출하지 않아도 request를 가져갈 수 있다.

IOPOLL은 일부 storage I/O에서 completion을 polling하는 방식이다. interrupt 기반 completion과 흐름이 다르기 때문에 완료 확인 경로가 달라진다.

이 문서에서는 두 기능의 내부 전체를 다루지 않는다. 중요한 점은 다음이다.

```text
polling mode에서는 누가 queue를 소비하고 완료를 밀어 넣는지 달라진다.
```

따라서 wakeup, sleep, busy polling, teardown 경로를 함께 확인해야 한다.

## overflow와 backpressure

CQ가 가득 차면 kernel은 completion을 즉시 CQ ring에 넣지 못할 수 있다. 이때 overflow list나 task work 경로가 관여할 수 있다.

확인할 질문은 다음이다.

- CQ 공간이 없을 때 completion은 어디에 보관되는가?
- overflow된 CQE는 user가 나중에 반드시 관찰할 수 있는가?
- ring teardown 중 overflow entry가 남지 않는가?
- multi-shot request가 CQ를 계속 채울 때 제한이 있는가?

CQ overflow는 단순 성능 문제가 아니라 completion loss와 lifetime 문제로 이어질 수 있다.

## 코드에서 확인할 것

SQ/CQ 관련 코드를 읽을 때는 다음을 찾는다.

- ring setup과 mmap offset 계산
- SQE를 읽어 request로 변환하는 위치
- head/tail load/store에 쓰이는 barrier
- CQE를 기록하는 함수
- CQ overflow 처리 경로
- SQPOLL/IOPOLL에서 queue를 소비하는 주체
- ring teardown 중 남은 completion 정리

## 보안 관점

SQ/CQ 관련 취약점은 user와 kernel이 공유하는 memory 경계에서 나온다.

- SQE field 검증 누락
- head/tail 계산 오류로 인한 out-of-bounds 접근
- memory ordering 누락으로 인한 partial SQE/CQE 관찰
- CQE 중복 기록 또는 누락
- overflow completion 정리 실패
- ring teardown 중 worker나 completion path가 ring을 계속 참조

검토할 질문은 다음과 같다.

1. user가 조작 가능한 SQE field는 모두 opcode별로 검증되는가?
2. SQ/CQ index 계산이 ring size 밖으로 나가지 않는가?
3. CQE는 request 하나의 완료 의미에 맞게 정확히 기록되는가?
4. ring이 닫히는 중에도 completion path가 안전하게 실패하거나 정리되는가?
5. polling mode에서 producer와 consumer가 바뀌어도 같은 queue 규칙이 유지되는가?

## 다른 문서와의 연결

- [request lifetime](request-lifetime.md): SQE는 내부 `io_kiocb` request로 변환된 뒤 별도 lifetime을 가진다.
- [workers / cancellation](workers-cancellation.md): worker 완료와 cancel 결과는 CQE로 user에게 전달된다.
- [registered buffers](registered-buffers.md): SQE가 고정 buffer index를 지정할 수 있다.
- [fixed files](fixed-files.md): SQE가 fd 대신 등록된 file slot을 사용할 수 있다.

## 기억할 문장

SQ/CQ는 io_uring의 빠른 통신 경로이며, 안전성은 shared ring의 head/tail, memory ordering, completion 기록 규칙이 맞을 때 유지된다.
