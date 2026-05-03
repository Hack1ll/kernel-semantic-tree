# ioctl

## 한 문장 정의

fd가 가리키는 kernel object에 subsystem별 command와 user-provided 구조체를 전달하는 확장 entry point.

## 핵심 질문

read/write로 표현하기 어려운 제어 작업을 커널은 어떻게 fd 하나에 묶어 전달받는가?

## 왜 필요한가

모든 kernel object가 byte stream처럼 read/write만 하면 충분한 것은 아니다. device 설정, VM 생성, GPU buffer 관리, terminal option 변경, network tunnel 설정처럼 object별로 다른 command가 필요하다.

ioctl은 이 다양한 command를 하나의 fd-based entry로 싣는다. 그래서 강력하지만 위험하다. command마다 입력 구조체, 권한, object lifetime, locking 규칙이 다르기 때문이다.

```text
fd
    -> struct file
    -> file_operations->unlocked_ioctl
    -> command switch
    -> command-specific copy / validation / state change
```

## ioctl을 볼 때의 중심 구조

ioctl은 syscall보다 더 subsystem-specific하다.

- fd는 대상 object를 고른다.
- command number는 수행할 operation을 고른다.
- argument pointer는 command-specific 구조체나 값을 전달한다.
- `file->private_data`는 driver/session object로 이어지는 경우가 많다.

따라서 ioctl handler를 읽을 때는 command switch 문을 API surface로 봐야 한다.

## 작동 흐름

1. user program이 `ioctl(fd, cmd, arg)`를 호출한다.
2. VFS가 fd를 `struct file`로 변환한다.
3. `file->f_op->unlocked_ioctl` 또는 compat handler가 호출된다.
4. handler가 `cmd`를 기준으로 operation을 선택한다.
5. command가 요구하는 구조체를 `copy_from_user()`로 가져온다.
6. size, flag, index, pointer, permission을 command별로 검증한다.
7. driver 또는 subsystem object 상태를 변경한다.
8. 필요한 결과를 `copy_to_user()`로 돌려준다.

## 대표 예시

GPU driver ioctl은 buffer object 생성, command submission, memory mapping 준비 같은 작업을 command별로 처리할 수 있다.

```text
ioctl(fd, CREATE_BUFFER, arg)
    -> struct file
    -> private_data: device context
    -> copy user structure
    -> size / flags 검증
    -> buffer object 생성
    -> handle 반환
```

이 예시를 읽을 때는 다음을 확인한다.

- `cmd`마다 기대하는 구조체 크기와 의미가 다른가?
- `private_data`가 올바른 type과 lifetime을 갖는가?
- compat ioctl에서 32-bit/64-bit 구조체 차이가 처리되는가?

## 핵심 용어

- `ioctl command`: 어떤 operation을 수행할지 나타내는 command number.
- `unlocked_ioctl`: 많은 driver와 subsystem이 구현하는 ioctl callback.
- `compat_ioctl`: 32-bit user space와 64-bit kernel 사이 구조체 차이를 처리하는 callback.
- `private_data`: open된 `struct file`에 붙어 driver 또는 subsystem session을 가리키는 pointer.
- `command structure`: command가 기대하는 user-provided input/output 구조체.
- `command switch`: `cmd` 값에 따라 처리 경로가 갈라지는 dispatch 지점.

## 다른 큰가지와의 연결

- [6. VFS / FD Model](../6-vfs-fd-model/README.md): ioctl은 fd와 `struct file`을 전제로 한다.
- [10. Device Drivers](../10-device-drivers/README.md): driver는 ioctl로 hardware-specific command를 받는다.
- [3. Memory Management](../3-memory-management/README.md): ioctl argument는 user pointer와 size 검증을 요구한다.
- [7. Permission Model](../7-permission-model/README.md): command별로 다른 권한이 필요할 수 있다.

## 헷갈리기 쉬운 부분

- ioctl 하나를 단일 API처럼 보는 것
- command별 구조체 크기와 field 의미를 구분하지 않는 것
- open 시점 권한으로 모든 command를 허용해도 된다고 생각하는 것
- `private_data` pointer가 항상 살아 있다고 가정하는 것

## 보안/취약점 관점

ioctl bug는 driver 취약점의 전형적인 입구다. 특히 command parser, compat path, object handle lookup, buffer size 검증에서 문제가 많이 나온다.

코드를 읽을 때는 다음 질문을 붙인다.

1. 이 command는 unprivileged fd에서 호출 가능한가?
2. `arg`가 pointer인지 integer인지 command별로 명확한가?
3. copy한 구조체의 size, count, index, flag가 모두 검증되는가?
4. `private_data`와 내부 object lifetime은 누가 보장하는가?
5. error path에서 생성한 object와 reference를 되돌리는가?

## 다음에 읽을 문서

- [copy_from_user / copy_to_user](copy-from-user-copy-to-user.md)
- [6. VFS / FD Model](../6-vfs-fd-model/README.md)
- [10. Device Drivers](../10-device-drivers/README.md)

## 기억할 문장

ioctl은 fd 하나 뒤에 여러 command-specific kernel API를 숨긴 강력한 확장 통로다.
