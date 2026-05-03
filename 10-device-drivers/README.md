# 10. Device Drivers

## 핵심 질문

커널은 제각각 다른 하드웨어를 어떻게 file operation, buffer, interrupt, DMA 규칙으로 감싸는가?

## 큰가지의 의미

driver는 하드웨어와 커널 API 사이의 adapter다. user space는 `/dev/...` 파일, ioctl, mmap 같은 형태로 장치를 만지지만, driver 내부에서는 hardware register, queue, DMA buffer, interrupt가 움직인다.

```text
user file operation
    -> driver object
    -> hardware state
    -> interrupt / DMA completion
    -> user-visible result
```

## 하위 문서의 역할

- [file_operations](file-operations.md): device file을 open/read/write/ioctl/mmap callback으로 연결한다.
- [ioctl](ioctl.md): 장치별 command와 user 구조체를 처리한다.
- [mmap](mmap.md): device memory나 DMA buffer를 user address space에 붙인다.
- [DMA](dma.md): 장치가 CPU를 거치지 않고 memory에 직접 접근하는 방식
- [interrupt](interrupt.md): hardware event가 kernel handler를 실행하게 하는 경로

## 이 장에서 특히 구분할 것

driver에는 세 종류의 입구가 섞인다.

```text
user entry     -> open / ioctl / mmap / read / write
hardware entry -> interrupt / DMA completion
kernel entry   -> probe / remove / suspend / worker
```

이 입구들이 같은 device object와 buffer를 동시에 만질 수 있다.

## 대표 흐름

```text
ioctl(fd, CMD, arg)
    -> copy_from_user
    -> command validation
    -> driver state update
    -> hardware register write
```

```text
DMA receive
    -> device writes memory
    -> interrupt
    -> driver completion path
    -> buffer ownership returns to CPU
```

## 다른 큰가지와의 연결

- [6. VFS / FD Model](../6-vfs-fd-model/README.md): device는 file operation으로 user space에 노출된다.
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): ioctl과 mmap은 강한 user input boundary다.
- [3. Memory Management](../3-memory-management/README.md): DMA buffer와 mmap은 memory rule과 연결된다.
- [5. Concurrency](../5-concurrency/README.md): interrupt, worker, file operation이 동시에 state를 만진다.

## 보안 관점

driver bug는 hardware 권한과 user input이 만나는 곳에서 생긴다.

- ioctl 구조체 size와 field 검증 누락
- `private_data` lifetime 오류
- device unplug와 file operation race
- DMA buffer length 또는 ownership 오류
- mmap offset 검증 누락
- interrupt context에서 sleep 가능한 작업 수행

## 읽고 나서 확인할 것

1. 이 경로는 user entry인가 hardware entry인가?
2. driver private object는 언제 생성되고 언제 해제되는가?
3. DMA나 mmap buffer는 누가 소유하고 있는가?
4. unplug, close, interrupt가 동시에 일어날 수 있는가?
5. user-provided command와 size는 모두 검증되는가?
