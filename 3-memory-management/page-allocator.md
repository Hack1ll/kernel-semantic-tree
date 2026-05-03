# page allocator

## 한 문장 정의

page allocator는 물리 메모리를 page 단위로 나누어 커널 내부와 user address space의 backing memory로 제공하는 가장 낮은 메모리 할당 계층이다.

## 왜 중요한가

커널의 많은 메모리 기능은 결국 page 위에 쌓인다. slab allocator도 page를 받아 object slot으로 나누고, anonymous memory도 page를 받아 process 주소 공간에 붙이며, file cache도 page 단위로 데이터를 관리한다.

page allocator를 읽을 때 핵심은 “어떤 타입의 객체인가”가 아니라 “몇 개의 연속된 page가 어떤 조건으로 필요한가”다.

```text
physical memory
    -> page frame
    -> struct page
    -> order 단위 할당
    -> caller가 목적에 맞게 사용
```

## page와 order

page는 커널이 물리 메모리를 관리하는 기본 단위다. 각 page frame에 대응하는 상태는 `struct page`로 추적된다.

큰 연속 메모리가 필요할 때는 order 단위로 요청한다.

```text
order 0 -> 1 page
order 1 -> 2 contiguous pages
order 2 -> 4 contiguous pages
order n -> 2^n contiguous pages
```

order가 커질수록 할당은 어려워진다. 충분한 free memory가 있어도 연속된 page가 없으면 high-order allocation은 실패할 수 있다.

## 할당 흐름

대표적인 흐름은 다음과 같다.

```text
alloc_pages(gfp, order)
    -> GFP flag로 sleep/reclaim 가능 여부 판단
    -> zone 선택
    -> buddy allocator free list 탐색
    -> 필요하면 reclaim 또는 compaction 시도
    -> struct page 반환
```

page allocator는 반환한 page를 어떤 C 구조체로 쓸지 알지 못한다. 그 의미는 호출자가 정한다.

## GFP flag와 context

page allocation은 호출 context에 강하게 묶인다. 같은 크기의 page 요청이라도 sleep 가능한 context인지에 따라 사용할 수 있는 flag가 달라진다.

- `GFP_KERNEL`: 일반 kernel context에서 사용하며 sleep과 reclaim이 가능할 수 있다.
- `GFP_ATOMIC`: interrupt, spinlock 보유 구간 등 sleep이 어려운 context에서 사용한다.
- `GFP_NOWAIT`: 기다리지 않고 가능한 범위에서만 시도한다.
- `__GFP_ZERO`: 반환된 page 내용을 0으로 초기화한다.

코드에서 allocation failure 처리가 있는지 반드시 봐야 한다. page allocation은 실패할 수 있다.

## zone과 NUMA

물리 메모리는 하나의 단순한 목록이 아니다. architecture와 장치 제약 때문에 zone으로 나뉘고, NUMA 시스템에서는 node별 locality도 고려된다.

- `ZONE_DMA`: 일부 장치가 접근 가능한 낮은 주소 영역
- `ZONE_NORMAL`: 일반 kernel mapping으로 접근 가능한 영역
- `ZONE_MOVABLE`: compaction과 migration에 유리한 영역

driver나 DMA 관련 코드를 읽을 때는 “아무 page나 받아도 되는가?”가 중요한 질문이 된다.

## page lifetime

page는 할당되었다고 항상 같은 의미를 유지하지 않는다. file cache page, anonymous page, slab backing page, DMA buffer page는 서로 다른 규칙을 갖는다.

따라서 page를 볼 때는 다음을 분리한다.

- 누가 page reference를 잡고 있는가?
- page가 page cache에 연결되어 있는가?
- page table에 mapping되어 있는가?
- reclaim 대상이 될 수 있는가?
- DMA 중인 page인가?

## 코드에서 확인할 것

1. 요청한 order가 꼭 필요한가?
2. 호출 context에서 사용한 GFP flag가 맞는가?
3. allocation failure 경로가 모든 reference와 lock을 정리하는가?
4. 반환된 page가 zeroed page라고 가정하고 있지 않은가?
5. page reference를 잡은 주체와 놓는 주체가 명확한가?
6. page가 user mapping, page cache, DMA, slab 중 어디에 연결되는가?

## 보안 관점

page allocator 문제는 대개 초기화, 수명, 연속성 가정에서 나온다.

- 초기화되지 않은 page 내용을 user space로 노출한다.
- page reference를 잘못 관리해 아직 사용하는 page를 재사용한다.
- high-order allocation 실패를 처리하지 않아 NULL dereference나 cleanup bug가 생긴다.
- DMA 가능한 page 조건을 잘못 판단해 장치가 잘못된 memory를 접근한다.
- page table에서 제거된 page를 다른 경로가 계속 참조한다.

## 다른 문서와의 연결

- [slab allocator](slab-allocator.md): page를 받아 작은 kernel object slot으로 나눈다.
- [virtual memory](virtual-memory.md): page가 user virtual address에 연결되는 방식
- [mmap](mmap.md): VMA와 page fault를 통해 page가 address space에 붙는 경로
- [4. Object Lifetime](../4-object-lifetime/README.md): page reference와 object lifetime
- [10. Device Drivers](../10-device-drivers/README.md): DMA와 device memory

## 기억할 문장

page allocator는 typed object를 주는 계층이 아니라, 물리 page라는 원재료를 조건에 맞게 할당하고 회수하는 계층이다.
