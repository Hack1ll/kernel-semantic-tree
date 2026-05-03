# file_operations

## 한 문장 정의

driver의 `file_operations`는 `/dev/...`로 열린 device fd에 대해 `open`, `read`, `write`, `ioctl`, `mmap`, `poll`, `release` 같은 VFS callback을 실제 driver code로 연결하는 table이다.

## 왜 중요한가

device driver는 hardware register나 queue를 user space에 직접 노출하지 않는다. user space는 device node를 file처럼 열고, VFS는 그 file에 연결된 driver callback을 호출한다.

```text
open("/dev/foo")
    -> inode / cdev
    -> struct file
    -> file->f_op
    -> driver callback
    -> driver device object
```

`file_operations`를 읽으면 user entry가 driver 내부 object로 들어가는 시작점을 볼 수 있다.

## device file과 driver object

device file은 user-visible entry이고, driver object는 실제 장치 상태를 담는 kernel object다.

driver callback은 보통 다음 중 하나로 device object를 찾는다.

- inode의 character device 정보
- `container_of()`로 얻는 driver private structure
- `file->private_data`
- bus-specific device object
- subsystem helper가 반환한 wrapper object

fd가 유효하다는 사실만으로 driver private object lifetime이 자동 보장되는 것은 아니다.

## open

`.open`은 device fd가 만들어질 때 driver가 per-open state를 준비하는 지점이다.

주요 작업은 다음과 같다.

- device object reference 획득
- exclusive open 여부 확인
- per-file session object 할당
- `file->private_data` 설정
- hardware power-up 또는 runtime PM reference 획득
- initial state 설정

open 실패 경로에서는 이미 얻은 reference와 allocation을 정확히 되돌려야 한다.

## read와 write

`.read`와 `.write`는 user buffer와 driver buffer 사이에서 data를 이동시킨다.

확인할 점은 다음이다.

- user buffer 접근은 copy helper를 통해 이루어지는가?
- requested count가 driver buffer 크기를 넘지 않는가?
- blocking/nonblocking mode가 올바르게 처리되는가?
- wait queue와 wakeup 조건이 맞는가?
- device remove 중 read/write가 안전하게 중단되는가?

read/write는 byte stream처럼 보여도 내부에서는 hardware queue, DMA buffer, interrupt completion과 연결될 수 있다.

## poll

`.poll`은 `select`, `poll`, `epoll`이 device readiness를 확인하는 경로다.

driver는 보통 wait queue를 등록하고, 현재 상태를 event mask로 반환한다.

중요한 점은 poll이 상태를 바꾸는 main path가 아니라 readiness를 관찰하는 path라는 것이다. readiness flag와 실제 read/write 가능 상태가 같은 lock 또는 atomic rule 아래에서 일관되어야 한다.

## release

`.release`는 마지막 file reference가 사라질 때 호출되는 정리 경로다.

대표 작업은 다음과 같다.

- `file->private_data` 회수
- pending read/write/ioctl 취소
- wait queue wakeup
- DMA mapping 해제
- interrupt/work/timer 정리
- device object reference 반환

`close(fd)`가 호출되었다고 해서 곧바로 `.release`가 실행된다고 단정하면 안 된다. dup된 fd나 in-flight operation이 file reference를 붙잡을 수 있다.

## remove와 file operation

device unplug 또는 driver remove는 user file operation과 동시에 일어날 수 있다.

안전한 driver는 보통 다음 순서를 만든다.

```text
remove 시작
    -> device state를 DEAD 또는 REMOVING으로 변경
    -> 새 operation 차단
    -> pending operation 깨우기
    -> interrupt/work/DMA 정리
    -> 마지막 reference 대기
    -> device object free
```

open file이 남아 있을 때 hardware만 먼저 사라질 수 있다는 점을 고려해야 한다.

## 코드에서 확인할 것

1. `.open`에서 어떤 driver object reference를 얻는가?
2. `file->private_data`는 어디서 설정되고 어떤 타입으로 해석되는가?
3. read/write가 user buffer size와 driver buffer size를 모두 검증하는가?
4. `.poll`이 관찰하는 readiness flag가 실제 queue 상태와 일치하는가?
5. `.release`가 pending work, timer, DMA, wait queue를 정리하는가?
6. remove path와 file operation이 동시에 실행되어도 object lifetime이 보장되는가?

## 보안 관점

driver `file_operations` 버그는 user fd와 hardware state가 만나는 지점에서 생긴다.

- `private_data` type confusion
- open 실패 경로 reference leak
- release와 ioctl/read/write race로 인한 UAF
- device remove 뒤 file operation이 hardware register에 접근
- blocking read/write wakeup 누락
- user count를 그대로 buffer copy에 사용해 overflow 발생

## 다른 문서와의 연결

- [ioctl](ioctl.md): command별 driver control path
- [mmap](mmap.md): device buffer를 user address space에 노출하는 callback
- [DMA](dma.md): read/write와 completion에서 쓰이는 buffer ownership
- [interrupt](interrupt.md): hardware completion과 file operation wakeup
- [6. VFS / FD Model / file_operations](../6-vfs-fd-model/file-operations.md): VFS 공통 dispatch 구조

## 기억할 문장

driver `file_operations`를 읽을 때 핵심은 “fd callback이 어떤 driver object를 찾고, open부터 release까지 그 object가 살아 있는가?”다.
