# 3. Memory Management

## 핵심 질문

커널은 메모리를 어떤 단위로 나누고, 어떤 주소와 권한 규칙으로 보호하며, 언제 재사용하는가?

## 큰가지의 의미

메모리는 단일 자원이 아니다. 커널은 같은 물리 메모리를 여러 층으로 나누어 다룬다.

```text
physical memory
    -> page
    -> virtual address
    -> VMA
    -> slab object
    -> user/kernel copy boundary
```

이 장은 주소 변환, page lifetime, object allocation, mapping permission을 하나의 큰가지로 묶는다.

## 하위 문서의 역할

- [page allocator](page-allocator.md): 물리 메모리를 page 단위로 할당하고 회수하는 바닥 계층
- [slab allocator](slab-allocator.md): 작은 kernel object를 빠르게 할당하고 재사용하는 계층
- [virtual memory](virtual-memory.md): process마다 독립 주소 공간이 있는 것처럼 보이게 하는 추상화
- [mmap](mmap.md): file, device, anonymous memory를 process 주소 공간에 붙이는 인터페이스
- [user/kernel memory separation](user-kernel-memory-separation.md): user pointer와 kernel pointer를 분리하는 보호 규칙

## 이 장에서 특히 구분할 것

가상 주소와 물리 주소는 다르다.  
page와 slab object도 다르다.  
user pointer와 kernel pointer도 다르다.

커널 memory bug를 읽을 때는 항상 어느 층의 문제인지 먼저 구분한다.

```text
address bug?
size / offset bug?
page lifetime bug?
slab object reuse bug?
permission bit bug?
copy_to_user leak?
```

## 대표 흐름

```text
kmalloc(sizeof(struct file))
    -> slab cache 선택
    -> object slot 할당
    -> 초기화
    -> 사용
    -> kfree()
    -> slot 재사용 가능
```

```text
mmap(fd, len, prot, flags, off)
    -> file permission 확인
    -> VMA 생성
    -> page fault 시 backing page 연결
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): user pointer는 복사 전까지 신뢰할 수 없다.
- [4. Object Lifetime](../4-object-lifetime/README.md): memory가 free된 뒤에도 pointer가 남을 수 있다.
- [5. Concurrency](../5-concurrency/README.md): page table, VMA, allocator state는 동시에 접근될 수 있다.
- [10. Device Drivers](../10-device-drivers/README.md): driver mmap과 DMA는 memory rule을 hardware와 연결한다.

## 보안 관점

메모리 취약점은 대개 경계와 수명 착각에서 나온다.

- heap overflow, OOB read/write
- UAF와 slab object reuse
- uninitialized memory leak
- user/kernel pointer confusion
- mmap offset/length overflow
- page refcount 오류

## 읽고 나서 확인할 것

1. 이 메모리는 page, VMA, slab object 중 무엇인가?
2. size, offset, alignment는 어디서 검증되는가?
3. 누가 이 memory를 소유하고 언제 해제하는가?
4. 해제 후 같은 slot이나 page가 어떤 object로 재사용될 수 있는가?
