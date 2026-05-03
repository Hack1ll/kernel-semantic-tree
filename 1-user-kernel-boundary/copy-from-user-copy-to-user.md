# copy_from_user / copy_to_user

## 한 문장 정의

user address space와 kernel address space 사이에서 데이터를 옮길 때 사용하는 보호된 copy boundary.

## 핵심 질문

커널은 왜 user pointer를 그냥 dereference하지 않고 전용 copy API를 사용해야 하는가?

## 왜 필요한가

user pointer는 kernel pointer가 아니다. 같은 숫자처럼 보여도 의미가 다르다. user memory는 접근 불가능할 수 있고, page fault를 일으킬 수 있고, 다른 thread가 값을 바꿀 수 있으며, kernel이 직접 믿으면 안 된다.

`copy_from_user()`와 `copy_to_user()`는 syscall entry가 아니라 memory trust boundary다.

```text
user pointer
    -> access / fault handling
    -> copy
    -> kernel buffer
    -> validation
    -> use
```

## copy_from_user와 copy_to_user의 차이

`copy_from_user()`는 user memory에서 kernel buffer로 값을 가져온다. 이후 검증과 사용은 kernel buffer 기준으로 해야 한다.

`copy_to_user()`는 kernel이 만든 결과를 user buffer로 돌려준다. 이때도 user buffer가 유효하지 않을 수 있으므로 실패를 처리해야 한다.

둘 다 “복사가 항상 성공한다”는 보장을 주지 않는다. partial copy와 fault 가능성을 고려해야 한다.

## 작동 흐름

1. syscall, ioctl, write callback 등이 user pointer와 size를 받는다.
2. 커널이 size overflow와 destination buffer 크기를 먼저 확인한다.
3. user range 접근 가능성을 확인하고 copy API를 호출한다.
4. copy 실패 또는 partial copy를 검사한다.
5. 복사된 kernel buffer의 field, length, flag를 검증한다.
6. 검증된 kernel buffer만 실제 operation에 사용한다.
7. 결과를 user space에 줄 때 `copy_to_user()` 실패도 처리한다.

## 대표 예시

ioctl handler가 user structure를 받는 경우를 보자.

```text
ioctl(fd, CMD, user_arg)
    -> copy_from_user(&karg, user_arg, sizeof(karg))
    -> karg.len / karg.flags 검증
    -> kmalloc(karg.len)
    -> subsystem object 변경
    -> copy_to_user(user_arg, &result, sizeof(result))
```

이 예시를 읽을 때는 다음을 확인한다.

- copy 전에 size overflow를 확인하는가?
- copy한 뒤 user memory를 다시 직접 읽지 않는가?
- copy 실패를 성공처럼 처리하지 않는가?

## 핵심 용어

- `user pointer`: user address space를 가리키는 주소. kernel pointer처럼 직접 사용하면 안 된다.
- `kernel buffer`: user data를 복사해 온 뒤 검증과 사용에 쓰는 kernel memory.
- `access_ok()`: user range가 접근 가능한 범위인지 확인하는 기초 검사.
- `partial copy`: 일부 byte만 복사되고 나머지는 실패한 상태.
- `TOCTOU`: 검사한 시점과 사용한 시점 사이에 값이나 object가 바뀌는 문제.
- `hardened usercopy`: kernel object에서 user copy가 허용된 범위를 강화하는 보호 기능.

## 다른 큰가지와의 연결

- [3. Memory Management](../3-memory-management/README.md): user/kernel memory separation과 직접 연결된다.
- [1. User/Kernel Boundary](README.md): syscall, ioctl, procfs/sysfs write path에서 반복된다.
- [10. Device Drivers](../10-device-drivers/README.md): driver ioctl은 user structure copy를 많이 사용한다.
- [5. Concurrency](../5-concurrency/README.md): copy 중 page fault나 sleep 가능성을 고려해야 한다.

## 헷갈리기 쉬운 부분

- `access_ok()`만 하면 user memory가 안전하다고 생각하는 것
- copy_from_user 이후에도 원래 user pointer를 다시 참조하는 것
- partial copy return value를 무시하는 것
- spinlock이나 atomic context에서 fault 가능한 user copy를 수행하는 것
- kernel stack 또는 heap padding을 copy_to_user로 노출하는 것

## 보안/취약점 관점

usercopy bug는 커널 memory corruption과 information leak으로 이어질 수 있다. 특히 size 계산, 구조체 padding, partial copy 처리, 검증 후 재참조가 핵심 위험이다.

코드를 읽을 때는 다음 질문을 붙인다.

1. user pointer를 직접 dereference하는 곳은 없는가?
2. copy size는 destination buffer보다 작거나 같은가?
3. copy 실패와 partial copy를 명확히 처리하는가?
4. 검증은 kernel buffer 기준으로 수행되는가?
5. user로 돌려주는 data에 uninitialized field나 kernel pointer가 섞이지 않는가?

## 다음에 읽을 문서

- [syscall](syscall.md)
- [ioctl](ioctl.md)
- [3. Memory Management](../3-memory-management/README.md)

## 기억할 문장

copy_from_user와 copy_to_user는 user memory를 kernel state로 바꾸거나 kernel result를 user memory로 내보내는 신뢰 경계다.
