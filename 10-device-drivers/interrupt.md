# interrupt

## 한 문장 정의

interrupt는 hardware event가 발생했을 때 CPU가 driver handler를 실행해 device status를 확인하고 완료 처리를 시작하게 하는 비동기 entry path다.

## 왜 중요한가

device driver에는 user syscall로 들어오는 path만 있는 것이 아니다. hardware는 I/O 완료, packet 수신, error, hotplug 같은 event를 interrupt로 알릴 수 있다.

```text
hardware event
    -> IRQ line / MSI-X
    -> interrupt handler
    -> status 확인
    -> acknowledge
    -> completion 또는 deferred work
```

interrupt는 user operation, DMA completion, device remove와 동시에 같은 driver state를 만질 수 있다.

## hardirq handler

일반 interrupt handler는 hardirq context에서 실행된다. 이 context에서는 sleep하면 안 된다.

hardirq handler에서 해야 할 일은 짧아야 한다.

- device status register 확인
- interrupt 원인 acknowledge
- 완료된 queue entry 확인
- 필요한 최소 state update
- bottom half, tasklet, workqueue, threaded IRQ 깨우기

긴 처리, blocking I/O, user memory copy, sleep 가능한 lock은 피해야 한다.

## threaded interrupt

일부 driver는 threaded IRQ를 사용한다. top half는 interrupt를 acknowledge하고, thread handler가 비교적 긴 처리를 맡을 수 있다.

```text
top half
    -> IRQ_WAKE_THREAD 반환

threaded handler
    -> sleep 가능한 작업 일부 수행 가능
    -> lock/context 규칙은 여전히 확인 필요
```

threaded IRQ라고 해서 모든 작업이 안전해지는 것은 아니다. device state와 remove path 동시성은 그대로 남는다.

## shared IRQ

IRQ line을 여러 device가 공유할 수 있다. shared IRQ handler는 interrupt가 자기 device에서 온 것인지 확인해야 한다.

확인할 점은 다음이다.

- status register로 자기 device interrupt인지 확인하는가?
- 자기 interrupt가 아니면 적절한 return 값을 주는가?
- shared handler에서 다른 device state를 건드리지 않는가?
- storm이나 stuck interrupt를 처리하는가?

## interrupt와 DMA completion

DMA operation은 completion interrupt와 함께 쓰이는 경우가 많다.

```text
driver queue DMA
    -> device processes buffer
    -> interrupt
    -> completion handler
    -> dma_unmap 또는 sync
    -> wake up waiting file operation
```

completion 전에 buffer ownership을 CPU로 되돌리면 안 된다. interrupt handler는 DMA API와 queue state를 정확한 순서로 다뤄야 한다.

## disable, synchronize, free

device remove나 shutdown에서는 interrupt가 더 이상 handler를 호출하지 않게 정리해야 한다.

자주 보는 API 의미는 다음과 같이 읽으면 된다.

- `disable_irq()`: 해당 IRQ delivery를 막거나 대기한다.
- `synchronize_irq()`: 이미 실행 중인 handler가 끝날 때까지 기다린다.
- `free_irq()`: IRQ handler 등록을 해제한다.

정확한 순서는 driver 구조에 따라 다르지만, handler가 참조하는 object를 free하기 전에 running handler와 pending work를 정리해야 한다.

## 코드에서 확인할 것

1. handler는 hardirq context인가, threaded handler인가?
2. hardirq에서 sleep 가능한 함수를 호출하지 않는가?
3. device status 확인과 interrupt acknowledge 순서가 맞는가?
4. DMA completion이면 map/unmap 또는 sync 순서가 맞는가?
5. remove path가 running interrupt handler를 기다리는가?
6. shared IRQ라면 자기 device interrupt인지 확인하는가?

## 보안 관점

interrupt 관련 버그는 비동기 race와 context 규칙 위반에서 생긴다.

- handler가 remove된 device object를 참조한다.
- hardirq context에서 sleep 가능한 작업을 수행한다.
- interrupt acknowledge 순서가 틀려 interrupt storm이 발생한다.
- DMA completion 전에 buffer를 CPU가 재사용한다.
- shared IRQ에서 자기 device 여부를 확인하지 않는다.
- handler와 ioctl/read/write가 같은 state를 lock 없이 동시에 바꾼다.

## 다른 문서와의 연결

- [DMA](dma.md): DMA completion과 buffer ownership 전환
- [file_operations](file-operations.md): interrupt가 read/write wait queue를 깨우는 경로
- [mmap](mmap.md): mmap된 buffer와 device event 동시성
- [5. Concurrency](../5-concurrency/README.md): hardirq context, spinlock, workqueue
- [4. Object Lifetime](../4-object-lifetime/README.md): handler가 참조하는 object lifetime

## 기억할 문장

interrupt를 읽을 때 핵심은 “handler가 어떤 context에서 실행되고, remove와 DMA completion 사이에서 driver object가 안전하게 살아 있는가?”다.
