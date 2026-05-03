# mmap

## 한 문장 정의

`mmap()`은 anonymous memory, file, device memory를 process의 virtual address range에 연결하는 syscall이며, 그 결과로 VMA가 만들어진다.

## 왜 중요한가

`read()`와 `write()`는 buffer를 통해 데이터를 복사하지만, `mmap()`은 address space 자체에 memory range를 등록한다. 이후 실제 page 연결은 page fault 시점에 일어날 수 있다.

`mmap()`을 읽을 때 핵심은 “어떤 fd 또는 backing object가 어떤 permission과 offset으로 어느 address range에 붙는가”다.

```text
mmap(addr, length, prot, flags, fd, offset)
    -> parameter 검증
    -> file/device/anonymous 종류 결정
    -> permission 확인
    -> VMA 생성
    -> page fault 시 backing page 연결
```

## mapping 종류

### anonymous mapping

file 없이 memory range를 만든다. heap 확장, thread stack, private writable memory에서 자주 보인다. page fault가 나면 새 anonymous page가 할당될 수 있다.

### file-backed mapping

fd가 가리키는 file의 내용을 address space에 mapping한다. read permission, write permission, shared/private 여부, file offset alignment가 중요하다.

### device mapping

driver가 device memory나 DMA buffer를 user address space에 노출할 때 사용한다. driver의 `file_operations->mmap`과 VMA operation이 핵심이다.

## prot와 flags

`prot`는 mapping의 접근 권한을 정한다.

- `PROT_READ`: 읽기 가능
- `PROT_WRITE`: 쓰기 가능
- `PROT_EXEC`: 실행 가능

`flags`는 mapping의 공유 방식과 성격을 정한다.

- `MAP_PRIVATE`: write 시 private copy가 생길 수 있다.
- `MAP_SHARED`: 변경이 backing object와 공유될 수 있다.
- `MAP_ANONYMOUS`: file 없이 anonymous memory를 만든다.
- `MAP_FIXED`: 지정 주소에 mapping을 강제로 배치한다.

permission 검사는 fd open mode, filesystem policy, LSM, architecture permission과 함께 봐야 한다.

## VMA 생성과 page fault

`mmap()`이 성공해도 모든 physical page가 즉시 준비되는 것은 아니다. 커널은 먼저 VMA를 만들고, 실제 page는 접근 시점에 준비할 수 있다.

```text
mmap 성공
    -> VMA range 등록
    -> user가 주소 접근
    -> page fault
    -> VMA의 backing object 확인
    -> page 준비 또는 device page 연결
    -> PTE 설치
```

따라서 `mmap()` 코드를 볼 때는 syscall path와 fault path를 같이 읽어야 한다.

## driver mmap

device driver는 `file_operations->mmap`을 통해 user process에 device buffer를 mapping할 수 있다.

주요 확인 지점은 다음과 같다.

- `vma->vm_start`, `vma->vm_end`로 계산한 length가 허용 범위 안인가?
- `vma->vm_pgoff`가 device buffer offset으로 안전하게 변환되는가?
- user에게 writable mapping을 허용해도 되는가?
- `remap_pfn_range()` 또는 `vm_insert_page()` 호출 조건이 맞는가?
- `vm_ops->open`, `close`, `fault`에서 buffer lifetime을 보장하는가?

driver mmap은 권한, page lifetime, device memory가 직접 만나는 경로다.

## unmap과 mprotect

mapping은 생성 후에도 바뀔 수 있다.

- `munmap()`: VMA range를 제거한다.
- `mprotect()`: mapping permission을 바꾼다.
- `mremap()`: mapping 크기나 위치를 바꿀 수 있다.
- process exit: 전체 address space가 정리된다.

따라서 VMA가 한 번 만들어졌다고 영구히 유지된다고 가정하면 안 된다.

## 코드에서 확인할 것

1. `length`와 `offset` 계산에서 overflow가 없는가?
2. `offset`이 page alignment를 만족하는가?
3. fd open mode와 requested `prot`가 일치하는가?
4. `MAP_SHARED`와 `MAP_PRIVATE`에 따른 write 의미가 맞는가?
5. driver buffer가 VMA보다 먼저 해제될 수 없는가?
6. `mprotect`, `munmap`, process exit 경로에서도 reference가 정리되는가?

## 보안 관점

`mmap()` 취약점은 address range, permission, backing object lifetime이 어긋날 때 생긴다.

- offset과 length overflow로 허용 범위를 넘어선다.
- read-only file을 writable shared mapping으로 노출한다.
- device memory를 user space에 과도하게 노출한다.
- VMA close 이후 driver buffer reference를 잘못 정리한다.
- page fault path에서 이미 해제된 object를 참조한다.
- `MAP_FIXED`로 기존 mapping을 덮는 효과를 고려하지 않는다.

## 다른 문서와의 연결

- [virtual memory](virtual-memory.md): `mmap()`이 만드는 VMA와 page table 관계
- [page allocator](page-allocator.md): anonymous page allocation
- [user/kernel memory separation](user-kernel-memory-separation.md): user address range 접근 규칙
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd와 `file_operations->mmap`
- [10. Device Drivers](../10-device-drivers/README.md): driver mmap과 device buffer

## 기억할 문장

`mmap()`은 데이터를 즉시 복사하는 syscall이 아니라, address range와 backing object의 관계를 VMA로 등록하는 syscall이다.
