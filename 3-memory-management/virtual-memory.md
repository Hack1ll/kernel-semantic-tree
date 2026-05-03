# virtual memory

## 한 문장 정의

virtual memory는 process가 사용하는 virtual address를 page table과 VMA 규칙을 통해 physical page와 permission으로 연결하는 메모리 추상화다.

## 왜 중요한가

프로그램이 보는 주소값은 물리 주소가 아니다. 같은 숫자의 virtual address라도 process가 다르면 다른 physical page로 변환될 수 있고, 같은 physical page라도 여러 virtual address에 mapping될 수 있다.

virtual memory를 이해할 때 핵심은 주소값 자체가 아니라 “그 주소가 어느 `mm_struct` 안에서 어떤 VMA와 page table entry로 해석되는가”다.

```text
process
    -> mm_struct
    -> vm_area_struct 목록 또는 tree
    -> page table
    -> physical page
```

## mm_struct

`mm_struct`는 process의 user address space 전체를 표현한다. process가 가진 memory map, page table, mmap lock, heap, stack, shared library mapping이 이 객체를 중심으로 관리된다.

thread는 `mm_struct`를 공유할 수 있다. 따라서 “process 하나당 주소 공간 하나”라고 단순히 고정하면 thread와 `clone()`을 읽을 때 혼란이 생긴다.

## VMA

`vm_area_struct`는 연속된 virtual address 범위와 permission을 표현한다.

예를 들어 다음 정보가 VMA에 들어간다.

- 시작 주소와 끝 주소
- read, write, execute permission
- private 또는 shared mapping
- anonymous mapping인지 file-backed mapping인지
- page fault가 났을 때 호출할 operation

VMA는 주소 범위의 정책을 설명한다. 실제 physical page는 page fault가 날 때 연결될 수 있다.

## page table

page table은 virtual address를 physical page와 permission bit로 변환한다. CPU는 memory access를 할 때 page table과 TLB를 통해 주소를 해석한다.

page table entry에는 대략 다음 의미가 담긴다.

- 이 virtual page가 present 상태인가?
- 어느 physical page frame을 가리키는가?
- user mode에서 접근 가능한가?
- write 가능한가?
- execute 가능한가?
- dirty/accessed 상태가 있는가?

VMA permission과 page table permission이 일관되어야 한다.

## page fault

process가 아직 page table에 연결되지 않은 주소를 접근하거나 permission에 맞지 않는 접근을 하면 page fault가 발생한다.

대표 흐름은 다음과 같다.

```text
user memory access
    -> page table lookup 실패 또는 permission fault
    -> kernel page fault handler
    -> mm_struct에서 VMA 검색
    -> 접근 권한 확인
    -> 필요한 page 준비
    -> page table entry 설치
    -> user execution 재개
```

유효하지 않은 주소거나 permission이 맞지 않으면 signal이 전달될 수 있다.

## copy-on-write

`fork()` 이후 parent와 child는 같은 physical page를 read-only로 공유할 수 있다. 둘 중 하나가 write를 시도하면 page fault가 발생하고, kernel이 새 page를 만들어 내용을 복사한 뒤 write 가능한 mapping으로 바꾼다.

이 규칙은 memory 절약에 중요하지만, refcount와 page table update가 정확해야 한다.

## 코드에서 확인할 것

1. 이 주소는 어느 `mm_struct` 기준으로 해석되는가?
2. 해당 address range를 덮는 VMA가 있는가?
3. VMA permission과 page table permission이 일치하는가?
4. page fault 경로에서 page reference를 올바르게 잡는가?
5. page table 변경 뒤 TLB flush가 필요한가?
6. COW 경로에서 shared page와 private page의 refcount가 정확한가?

## 보안 관점

virtual memory 버그는 주소 해석과 permission 불일치에서 나온다.

- user address를 kernel pointer로 잘못 해석한다.
- write가 금지된 VMA에 writable PTE를 설치한다.
- executable permission을 잘못 설정한다.
- COW refcount를 틀려 shared page가 오염된다.
- page table update와 TLB flush 순서가 맞지 않는다.
- VMA lookup 뒤 concurrent unmap을 고려하지 않는다.

## 다른 문서와의 연결

- [mmap](mmap.md): VMA를 생성하는 대표 user interface
- [page allocator](page-allocator.md): page fault가 새 physical page를 필요로 할 때
- [user/kernel memory separation](user-kernel-memory-separation.md): user address와 kernel address의 접근 규칙
- [2. Process / Task](../2-process-task/README.md): `task_struct -> mm`
- [5. Concurrency](../5-concurrency/README.md): mmap lock, page table lock, TLB shootdown

## 기억할 문장

virtual memory에서 주소값은 절대적인 위치가 아니라, `mm_struct`, VMA, page table을 거쳐 해석되는 이름이다.
