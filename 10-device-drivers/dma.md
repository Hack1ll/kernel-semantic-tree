# DMA

## 한 문장 정의

DMA는 device가 CPU load/store를 거치지 않고 system memory를 직접 읽거나 쓰게 하는 mechanism이며, driver는 DMA API로 buffer address, direction, ownership, cache coherency를 관리한다.

## 왜 중요한가

device가 memory에 직접 접근하면 CPU와 driver만 memory를 바꾼다는 가정이 깨진다. driver는 device가 접근해도 되는 buffer만 DMA에 넘기고, device가 쓰는 동안 CPU가 같은 buffer를 잘못 만지지 않게 해야 한다.

```text
CPU owns buffer
    -> dma_map_*
    -> device owns or accesses buffer
    -> interrupt / completion
    -> dma_unmap_* or dma_sync_*
    -> CPU owns buffer again
```

DMA는 memory lifetime, cache coherency, device trust boundary가 만나는 영역이다.

## coherent DMA와 streaming DMA

DMA buffer는 크게 두 방식으로 다룬다.

### coherent DMA

CPU와 device가 같은 buffer를 볼 때 cache coherency가 유지되도록 할당한다.

대표 API:

- `dma_alloc_coherent()`
- `dma_free_coherent()`

descriptor ring처럼 CPU와 device가 반복적으로 공유하는 구조에 자주 쓰인다.

### streaming DMA

일반 memory buffer를 일정 기간 device에 넘긴다.

대표 API:

- `dma_map_single()`
- `dma_unmap_single()`
- `dma_map_page()`
- `dma_map_sg()`
- `dma_unmap_sg()`

map과 unmap 사이의 ownership 규칙을 지켜야 한다.

## direction

DMA direction은 device가 memory를 읽는지 쓰는지를 나타낸다.

- `DMA_TO_DEVICE`: device가 memory에서 읽는다.
- `DMA_FROM_DEVICE`: device가 memory에 쓴다.
- `DMA_BIDIRECTIONAL`: 양방향 접근 가능

direction이 틀리면 cache sync가 잘못되어 device가 stale data를 읽거나 CPU가 오래된 값을 볼 수 있다.

## ownership

streaming DMA에서 buffer ownership은 매우 중요하다.

```text
CPU fills buffer
    -> dma_map_single(..., DMA_TO_DEVICE)
    -> device reads buffer
    -> completion
    -> dma_unmap_single()
    -> CPU may reuse/free buffer
```

`DMA_FROM_DEVICE` 방향에서는 device가 쓰는 동안 CPU가 buffer 내용을 신뢰하면 안 된다. completion과 sync 이후에 읽어야 한다.

## IOMMU와 DMA mask

device가 접근할 수 있는 address 범위는 제한될 수 있다. driver는 DMA mask와 IOMMU mapping을 고려해야 한다.

확인할 점은 다음이다.

- `dma_set_mask_and_coherent()` 같은 설정이 성공했는가?
- device가 32-bit DMA만 가능한가?
- IOMMU가 device별 address translation을 제공하는가?
- user-controlled physical address를 device에 직접 주지 않는가?

IOMMU가 있어도 driver가 잘못된 buffer를 mapping하면 device는 그 buffer에 접근할 수 있다.

## scatter-gather

큰 buffer는 여러 memory segment로 나뉘어 device에 전달될 수 있다. 이때 scatter-gather list를 사용한다.

중요한 점은 다음이다.

- `dma_map_sg()`가 반환한 mapped entry 수를 사용해야 한다.
- original sg entry 수와 mapped entry 수가 다를 수 있다.
- 각 segment length가 device limit을 넘지 않는가?
- unmap 시 같은 direction과 mapped entry 정보를 사용한다.

## 코드에서 확인할 것

1. DMA buffer는 coherent인가, streaming인가?
2. direction이 device 접근 방향과 맞는가?
3. map failure를 확인하는가?
4. map과 unmap이 모든 success/error path에서 짝을 이루는가?
5. device가 쓰는 동안 CPU가 buffer를 읽거나 free하지 않는가?
6. DMA length, sg entry count, device address limit이 모두 검증되는가?

## 보안 관점

DMA bug는 device가 잘못된 memory를 읽거나 쓰는 문제로 이어진다.

- user-controlled length로 DMA buffer boundary를 넘는다.
- DMA map failure를 무시하고 invalid DMA address를 device에 준다.
- device completion 전에 buffer를 free해 memory corruption이 생긴다.
- direction이 틀려 stale data나 data leak이 생긴다.
- sg mapped entry 수를 잘못 사용해 OOB DMA가 발생한다.
- untrusted device에 민감한 kernel memory를 mapping한다.

## 다른 문서와의 연결

- [mmap](mmap.md): DMA buffer를 user address space에 노출하는 경우
- [interrupt](interrupt.md): DMA completion을 알리는 hardware event
- [file_operations](file-operations.md): read/write/ioctl에서 DMA queue를 조작하는 경로
- [3. Memory Management](../3-memory-management/README.md): page, cache, buffer lifetime
- [10. Device Drivers](README.md): driver entry와 hardware state 전체 흐름

## 기억할 문장

DMA를 읽을 때 핵심은 “이 buffer를 지금 CPU가 소유하는가, device가 접근 중인가, 그리고 그 전환이 DMA API로 정확히 표시되는가?”다.
