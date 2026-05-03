# registered buffers

## 한 문장 정의

`registered buffers`는 반복 I/O에서 매번 user memory를 해석하고 pin하지 않도록, io_uring context에 미리 등록해 둔 user buffer 집합이다.

## 왜 중요한가

일반 read/write request는 user pointer와 length를 받을 때마다 memory 접근 가능성, iov, page pinning 등을 처리해야 한다. registered buffer는 이 비용을 줄이기 위해 user memory 범위를 먼저 등록하고, request에서는 buffer index로 재사용한다.

이 문서의 줄기는 이것이다.

```text
registered buffer는 user memory를 kernel I/O 경로에서 반복 사용하기 위해 pin과 lifetime을 미리 관리하는 기능이다.
```

따라서 핵심은 빠른 접근보다 “이 user memory가 I/O 완료 전까지 실제로 살아 있고 같은 의미를 유지하는가”다.

## 등록 흐름

대표 흐름은 다음과 같다.

```text
io_uring_register(IORING_REGISTER_BUFFERS)
    -> user iovec 배열 복사
    -> address/length 검증
    -> page pin 또는 buffer state 생성
    -> ring context의 buffer table에 저장
```

이후 request는 SQE에서 fixed buffer 사용을 표시하고 buffer index를 지정할 수 있다.

```text
SQE fixed buffer index
    -> registered buffer table lookup
    -> request가 buffer reference 확보
    -> I/O 수행
    -> completion 후 request cleanup
```

## user memory pinning

registered buffer는 user address range를 다룬다. kernel은 이 범위가 I/O 중 사라지거나 다른 page로 바뀌지 않도록 page pinning을 사용한다.

확인할 항목은 다음이다.

- address와 length가 user address range 안에 있는가?
- page boundary를 넘는 범위가 정확히 계산되는가?
- `length == 0` 같은 특수 값이 허용되는가?
- pin된 page 수와 memory accounting이 맞는가?
- read/write 방향에 따른 page dirty 처리가 필요한가?
- process mm lifetime과 buffer table lifetime이 분리되어 있는가?

registered buffer는 `mmap` 문서의 VMA 개념과 연결되지만, 여기서는 io_uring이 I/O용 user page를 어떻게 붙잡는지만 다룬다.

## fixed buffer 사용

request에서 fixed buffer를 쓰면 SQE의 user pointer 대신 등록된 buffer index가 기준이 된다.

확인할 규칙은 다음이다.

- index가 buffer table 범위 안에 있는가?
- request offset과 length가 등록된 buffer 범위 안에 있는가?
- partial I/O에서 실제 사용 범위가 계산된 범위를 넘지 않는가?
- buffer가 unregister 중이면 새 request가 붙잡을 수 없는가?
- 이미 붙잡은 request는 unregister와 독립적으로 completion까지 안전한가?

index 검증은 단순 array bounds check로 끝나지 않는다. 해당 slot의 상태와 request별 사용 범위를 함께 봐야 한다.

## provided buffer와의 차이

io_uring에는 provided buffer 또는 buffer selection 계열 기능도 있다. 이 기능은 kernel이 completion 때 어떤 buffer가 사용되었는지 CQE flags로 알려줄 수 있다.

registered buffer와 구분할 점은 다음이다.

- registered buffer: 고정된 user memory 범위를 미리 등록하고 request에서 index로 지정
- provided buffer: receive 계열에서 kernel이 buffer group에서 사용할 buffer를 선택할 수 있음

두 기능 모두 buffer lifetime과 CQE flags를 다루지만, 이 문서는 fixed buffer registration을 중심으로 한다.

## unregister와 update

buffer는 등록만 하는 것이 아니라 해제되거나 갱신될 수 있다.

위험한 흐름은 다음이다.

```text
request A가 buffer slot 3 사용 중
    -> user가 buffer unregister 요청
    -> request A completion 전 page가 unpin되면 안 됨
```

따라서 unregister는 in-flight request와의 관계를 고려해야 한다.

확인할 질문은 다음이다.

- unregister가 진행 중인 slot에 새 request가 붙는가?
- 이미 slot을 잡은 request는 별도 reference를 가지는가?
- update 실패 시 기존 buffer 상태가 유지되는가?
- ring exit 중 남은 buffer reference가 모두 정리되는가?

## DMA와의 관계

registered buffer가 곧바로 device DMA에 안전하다는 뜻은 아니다. 일부 I/O 경로는 file subsystem, block layer, networking stack을 지나며 각 계층의 DMA 규칙을 따른다.

io_uring 관점에서 확인할 것은 다음이다.

- user page가 I/O 기간 동안 움직이지 않는가?
- write I/O 뒤 dirty 처리나 cache coherency가 필요한가?
- long-term pin이 memory management에 주는 영향이 고려되는가?
- device나 filesystem 경로가 pinned user page를 올바르게 처리하는가?

DMA 자체의 세부 규칙은 [10. Device Drivers - DMA](../10-device-drivers/dma.md)에서 다룬다.

## 코드에서 확인할 것

registered buffer 코드를 읽을 때는 다음을 본다.

- register syscall에서 user iovec을 복사하는 위치
- address/length/page count 계산
- page pin과 unpin 경로
- buffer table slot 상태 전환
- request가 buffer를 lookup하고 붙잡는 지점
- unregister/update 중 in-flight request와 동기화하는 방식
- completion cleanup에서 buffer reference를 내려놓는 위치

## 보안 관점

registered buffer 관련 취약점은 주로 user memory lifetime과 범위 계산에서 나온다.

- address + length overflow
- page count 계산 오류
- buffer index 검증 누락
- request length가 등록 범위를 넘는 문제
- unregister와 in-flight I/O의 race
- pin/unpin 불균형
- error path에서 일부 page만 unpin
- provided buffer와 fixed buffer 의미 혼동

검토할 질문은 다음과 같다.

1. user가 준 address와 length는 overflow 없이 page 범위로 변환되는가?
2. request가 사용하는 offset과 length는 등록된 buffer 안에 머무르는가?
3. unregister가 완료되기 전 in-flight request의 buffer reference가 안전한가?
4. pin 실패 중간 경로에서 이미 pin된 page가 모두 해제되는가?
5. completion 이후 buffer cleanup이 request lifetime과 정확히 맞물리는가?

## 다른 문서와의 연결

- [request lifetime](request-lifetime.md): request가 buffer reference를 언제 얻고 언제 내려놓는지 연결된다.
- [SQ / CQ](sq-cq.md): SQE는 fixed buffer index를 전달하고, CQE는 buffer selection 결과를 알릴 수 있다.
- [workers / cancellation](workers-cancellation.md): cancel되어도 buffer cleanup은 정확히 실행되어야 한다.
- [3. Memory Management](../3-memory-management/README.md): user page pinning과 가상 메모리 lifetime을 함께 봐야 한다.

## 기억할 문장

registered buffer는 user memory를 빠르게 쓰기 위한 기능이지만, 안전성은 pinning, 범위 검증, unregister와 in-flight request의 관계에서 결정된다.
