# mmap

## 한 문장 정의

driver `mmap`은 device memory, DMA buffer, ring buffer 같은 driver-managed memory를 user process의 virtual address range에 연결하는 file operation이다.

## 왜 중요한가

일부 device는 read/write copy 대신 user process가 buffer를 직접 mapping해서 사용한다. 이 방식은 성능상 유리할 수 있지만, driver buffer lifetime과 user VMA lifetime이 강하게 묶인다.

```text
mmap(fd, length, prot, flags, offset)
    -> file->f_op->mmap
    -> driver buffer 선택
    -> VMA 설정
    -> page 또는 PFN mapping
    -> user access
```

driver mmap은 memory management와 device lifetime이 직접 만나는 지점이다.

## VMA와 driver buffer

driver mmap callback은 `vm_area_struct`를 받아 user address range를 설정한다.

주요 field는 다음이다.

- `vma->vm_start`: user mapping 시작 주소
- `vma->vm_end`: user mapping 끝 주소
- `vma->vm_pgoff`: user가 넘긴 page offset
- `vma->vm_flags`: mapping 성격
- `vma->vm_ops`: fault/open/close callback
- `vma->vm_private_data`: driver private mapping state

driver는 이 VMA가 어떤 buffer나 device region을 가리키는지 명확히 정해야 한다.

## offset과 length 검증

`mmap`에서 가장 먼저 볼 것은 range 검증이다.

확인할 점은 다음이다.

- `vm_end - vm_start`가 허용된 mapping 크기 안에 있는가?
- `vm_pgoff`가 page 단위 offset으로 안전하게 해석되는가?
- offset + length 계산에서 overflow가 없는가?
- user가 device memory 전체를 mapping하지 못하게 제한하는가?
- requested protection이 driver policy와 맞는가?

offset 검증이 빠지면 user가 의도하지 않은 device memory나 kernel memory에 접근할 수 있다.

## mapping 방식

driver는 buffer 성격에 따라 다른 API를 쓸 수 있다.

- `remap_pfn_range()`: physical frame number range를 VMA에 mapping
- `vm_insert_page()`: 특정 page를 fault path나 setup path에서 삽입
- `dma_mmap_coherent()`: coherent DMA buffer를 user space에 mapping
- custom `vm_ops->fault`: page fault 시점에 page를 제공

API 선택은 buffer 종류, cache attribute, device DMA 제약과 맞아야 한다.

## vm_ops lifetime

VMA는 file descriptor close 이후에도 남을 수 있다. user process가 mapping을 유지하면 driver buffer도 그 기간 동안 살아 있어야 한다.

그래서 driver는 `vm_ops`를 통해 mapping lifetime을 추적할 수 있다.

- `.open`: VMA가 복제되거나 새 reference가 필요할 때 호출
- `.close`: VMA가 사라질 때 mapping reference 반환
- `.fault`: page fault 때 page 제공

`file->private_data`가 release되었다고 VMA가 함께 사라진다고 가정하면 안 된다.

## cache attribute

device memory와 DMA buffer는 CPU cache 정책이 중요하다.

확인할 점은 다음이다.

- memory가 cacheable인지 uncached인지 write-combined인지
- CPU와 device가 같은 buffer를 동시에 볼 때 coherency가 보장되는지
- DMA coherent buffer를 일반 memory처럼 mapping하지 않는지
- architecture별 pgprot 설정이 필요한지

cache attribute가 틀리면 data corruption이나 stale data 문제가 생길 수 있다.

## 코드에서 확인할 것

1. `vm_pgoff`, length, offset + length가 모두 범위 안에 있는가?
2. user requested protection이 driver가 허용한 permission과 맞는가?
3. mapping할 buffer가 file close 이후에도 VMA lifetime 동안 살아 있는가?
4. device remove 중 mmap된 buffer 접근을 어떻게 막는가?
5. cache attribute와 DMA coherency가 buffer 성격과 맞는가?
6. `vm_ops->fault/open/close`에서 reference가 정확히 맞는가?

## 보안 관점

driver mmap 버그는 user address space에 과도한 memory를 노출할 때 생긴다.

- offset 검증 누락으로 device memory 범위를 넘어 mapping한다.
- length overflow로 작은 검증을 통과한 뒤 큰 range를 mapping한다.
- read-only여야 하는 buffer를 writable로 mapping한다.
- file release 후 VMA가 driver buffer를 계속 참조해 UAF가 생긴다.
- device remove와 page fault가 race를 일으킨다.
- cache attribute 불일치로 stale data나 corruption이 발생한다.

## 다른 문서와의 연결

- [DMA](dma.md): DMA buffer를 user space에 mapping하는 경우
- [file_operations](file-operations.md): `.mmap` callback이 연결되는 dispatch table
- [ioctl](ioctl.md): ioctl로 buffer를 등록한 뒤 mmap하는 driver 구조
- [3. Memory Management / mmap](../3-memory-management/mmap.md): 일반 VMA와 mmap 개념
- [4. Object Lifetime](../4-object-lifetime/README.md): VMA와 buffer reference lifetime

## 기억할 문장

driver mmap을 읽을 때 핵심은 “이 VMA가 어떤 device buffer를 어느 범위와 permission으로 노출하고, 그 buffer가 VMA보다 오래 살아 있는가?”다.
