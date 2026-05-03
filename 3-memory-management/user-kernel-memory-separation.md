# user/kernel memory separation

## 한 문장 정의

user/kernel memory separation은 user address와 kernel address를 분리하고, 커널이 user memory를 다룰 때 전용 copy helper와 permission check를 거치게 하는 보호 규칙이다.

## 왜 중요한가

커널은 모든 메모리에 접근할 수 있는 권한 있는 코드다. 그래서 user가 넘긴 pointer를 kernel pointer와 같은 방식으로 dereference하면 안 된다.

이 문서의 핵심은 “주소값이 있다”와 “커널이 그 주소를 안전하게 읽거나 쓸 수 있다”를 분리하는 것이다.

```text
user pointer
    -> user address space의 주소
    -> access_ok / copy helper 필요
    -> page fault 또는 partial copy 가능

kernel pointer
    -> kernel address space의 주소
    -> 일반 dereference 가능
    -> user에게 직접 노출하면 안 됨
```

## user pointer

syscall, ioctl, netlink 일부 경로에서 user program은 kernel에 pointer 값을 넘긴다. 이 pointer는 user address space 기준의 주소다.

커널 코드에서는 보통 `__user` annotation으로 user pointer를 표시한다.

```c
char __user *buf;
```

`__user`가 붙은 pointer는 일반 `*buf` 방식으로 읽거나 쓰면 안 된다. `copy_from_user()`, `copy_to_user()`, `get_user()`, `put_user()` 같은 helper를 사용해야 한다.

## copy_from_user

`copy_from_user()`는 user memory에서 kernel buffer로 데이터를 복사한다.

```text
user buffer
    -> copy_from_user()
    -> kernel buffer
    -> kernel parser 또는 state update
```

확인할 점은 다음과 같다.

- return value를 확인하는가?
- 복사 대상 kernel buffer 크기가 충분한가?
- 복사 후 구조체 field를 모두 검증하는가?
- user가 복사 전후에 원본 memory를 바꿀 수 있음을 고려하는가?

검증은 copy 이후 kernel buffer 기준으로 해야 한다.

## copy_to_user

`copy_to_user()`는 kernel memory에서 user buffer로 데이터를 복사한다.

```text
kernel result
    -> field 초기화
    -> copy_to_user()
    -> user buffer
```

여기서 가장 위험한 실수는 초기화되지 않은 kernel memory를 user space로 내보내는 것이다. padding, union field, error path에서 덜 채운 구조체도 정보 노출이 될 수 있다.

## access_ok와 partial copy

`access_ok()`는 user pointer range가 user address로 보이는지 검사한다. 하지만 그것만으로 실제 복사가 항상 성공한다고 보장하지 않는다.

copy helper는 page fault, unmapped page, permission 문제 때문에 일부만 복사하고 실패할 수 있다. 따라서 return value를 무시하면 안 된다.

```text
access_ok 통과
    -> copy 중 page fault 가능
    -> partial copy 가능
    -> return value 확인 필요
```

## TOCTOU

user memory는 커널이 한 번 확인한 뒤에도 user process가 바꿀 수 있다. 그래서 같은 user pointer에서 field를 여러 번 직접 읽으면 검사한 값과 사용한 값이 달라질 수 있다.

안전한 기본 형태는 다음과 같다.

```text
user pointer
    -> kernel buffer로 한 번 복사
    -> kernel buffer 검증
    -> 검증한 kernel buffer 값으로 동작
```

큰 buffer를 chunk 단위로 처리해야 한다면 각 chunk의 length, offset, state transition을 따로 검증해야 한다.

## kernel pointer 노출

kernel pointer 값은 user에게 그대로 노출되면 안 된다. pointer leak은 KASLR 우회나 exploit 안정화에 쓰일 수 있다.

주의할 경로는 다음과 같다.

- `copy_to_user()`로 구조체 전체를 내보낼 때 padding에 pointer 값이 남아 있는 경우
- debugfs, procfs, sysfs 출력에서 kernel address를 출력하는 경우
- error message나 trace 경로가 pointer 값을 노출하는 경우
- uninitialized stack 또는 heap memory가 user로 복사되는 경우

## 코드에서 확인할 것

1. user pointer에 `__user` 의미가 표시되어 있는가?
2. user pointer를 직접 dereference하지 않는가?
3. `copy_from_user()`와 `copy_to_user()` return value를 확인하는가?
4. user input은 kernel buffer로 복사한 뒤 검증하는가?
5. `copy_to_user()` 전에 모든 field와 padding이 초기화되는가?
6. pointer 값이나 kernel address가 user에게 노출되지 않는가?

## 보안 관점

user/kernel memory separation이 깨지면 취약점 영향이 커진다.

- user pointer를 kernel pointer로 사용해 arbitrary read/write가 생긴다.
- `copy_from_user()` 실패를 무시해 덜 복사된 구조체를 사용한다.
- validation과 use 사이에 user memory가 바뀌어 TOCTOU가 생긴다.
- `copy_to_user()`가 uninitialized kernel memory를 노출한다.
- kernel address leak으로 KASLR 보호가 약해진다.

## 다른 문서와의 연결

- [virtual memory](virtual-memory.md): user address가 `mm_struct`와 page table로 해석되는 방식
- [mmap](mmap.md): user address range를 만드는 대표 인터페이스
- [slab allocator](slab-allocator.md): kernel heap object가 user로 복사될 때의 정보 노출 위험
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syscall, ioctl, copy helper
- [7. Permission Model](../7-permission-model/README.md): user request를 권한 기준으로 검증하는 위치

## 기억할 문장

user pointer는 kernel pointer가 아니며, 커널은 user memory를 복사하고 검증한 값만 신뢰해야 한다.
