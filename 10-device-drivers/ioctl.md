# ioctl

## 한 문장 정의

driver `ioctl`은 fd, command, user argument를 받아 read/write만으로 표현하기 어려운 device-specific 제어 작업을 수행하는 file operation이다.

## 핵심 질문

user program이 장치별 설정, 상태 조회, buffer 등록, 동작 시작 같은 작업을 요청할 때 driver는 command별 입력을 어떻게 검증하고 device state를 바꾸는가?

## 왜 중요한가

`ioctl()`은 driver마다 command와 argument 형식이 다르다. 그래서 공통 syscall 모양은 단순하지만, 실제 위험은 driver가 정의한 command parser와 state update에 있다.

```text
ioctl(fd, cmd, arg)
    -> fd lookup
    -> struct file
    -> file->f_op->unlocked_ioctl
    -> driver command handler
    -> device/session state update
```

`cmd`와 `arg`는 user가 직접 제어하는 입력이다. command별로 별도 검증이 필요하다.

## fd와 private_data

VFS는 fd를 `struct file`로 바꾸고 driver의 ioctl callback을 호출한다. driver는 보통 `file->private_data`에서 device object나 per-open session object를 꺼낸다.

확인할 점은 다음이다.

- `private_data`가 `.open`에서 확실히 설정되는가?
- ioctl callback에서 예상 타입으로만 해석되는가?
- `.release`나 device remove와 동시에 사라질 수 없는가?
- per-device state와 per-open state를 혼동하지 않는가?

## command 번호

ioctl command는 보통 방향, type, number, size 정보를 담는 encoding을 쓴다.

- `_IO`: data copy가 없는 command
- `_IOR`: kernel에서 user로 data를 돌려주는 command
- `_IOW`: user에서 kernel로 data를 받는 command
- `_IOWR`: 양방향 data 이동 command

하지만 command encoding만 믿으면 안 된다. 실제 handler가 기대하는 structure size와 copy 방향을 코드에서 확인해야 한다.

## user argument copy

`arg`가 user pointer라면 직접 dereference하면 안 된다.

기본 흐름은 다음과 같다.

```text
arg user pointer
    -> copy_from_user()
    -> kernel local structure
    -> field validation
    -> state update
    -> copy_to_user() if needed
```

검증은 kernel buffer로 복사한 뒤 수행해야 한다. user memory를 여러 번 읽으면 검사한 값과 사용한 값이 달라질 수 있다.

## command별 검증

ioctl handler는 command마다 기대하는 구조체와 의미가 다르다.

각 command에서 확인할 항목은 다음이다.

- structure size가 command와 맞는가?
- index가 table 범위 안에 있는가?
- length 계산에서 overflow가 없는가?
- flag 조합이 허용된 값인가?
- user pointer를 다시 포함하는 nested pointer가 있는가?
- command 실행에 별도 capability가 필요한가?
- 실패 시 이미 만든 object와 reference를 정리하는가?

하나의 ioctl handler 안에서도 command별 위험은 서로 다르다.

## compat ioctl

64-bit kernel에서 32-bit user program을 지원하면 `compat_ioctl` 경로가 필요할 수 있다.

주의할 점은 다음이다.

- pointer size가 다르다.
- structure padding과 alignment가 다르다.
- native ioctl과 같은 검증을 수행해야 한다.
- compat structure를 native structure로 변환할 때 field 손실이 없어야 한다.

compat path를 별도 구현하면서 검증이 약해지는 경우가 있다.

## 상태 변경 순서

권한 검사와 input 검증은 hardware state나 driver object를 바꾸기 전에 끝나야 한다.

```text
cmd 확인
    -> user data copy
    -> field validation
    -> permission check
    -> lock 획득
    -> state update
    -> hardware operation
```

실제 순서는 driver 성격에 따라 달라질 수 있지만, 검증되지 않은 값으로 allocation, array access, register write를 하면 안 된다.

## 코드에서 확인할 것

1. `cmd`마다 기대하는 `arg` 형식이 명확한가?
2. user data를 kernel buffer로 복사한 뒤 검증하는가?
3. size, index, offset, flag 계산에서 overflow가 없는가?
4. command별 capability 또는 ownership check가 필요한가?
5. compat ioctl이 native ioctl과 같은 수준으로 검증되는가?
6. close, remove, interrupt, mmap과 동시에 실행되어도 driver state가 안전한가?

## 보안 관점

driver ioctl은 user-controlled command가 kernel driver state를 직접 바꾸는 경로다.

- command별 size 검증 누락
- integer overflow로 잘못된 allocation 또는 copy size 계산
- index 검증 누락으로 out-of-bounds 접근
- `copy_from_user()` 실패 처리 누락
- compat ioctl 구조체 변환 오류
- `private_data` lifetime 오류
- command별 권한 검사 누락
- error path에서 object나 DMA mapping 정리 누락

## 다른 문서와의 연결

- [file_operations](file-operations.md): ioctl callback이 연결되는 dispatch table
- [mmap](mmap.md): ioctl로 등록한 buffer를 mmap에서 사용할 수 있는 경우
- [DMA](dma.md): ioctl이 DMA buffer 등록이나 queue submission을 수행하는 경우
- [1. User/Kernel Boundary / ioctl](../1-user-kernel-boundary/ioctl.md): syscall boundary로서의 ioctl
- [1. User/Kernel Boundary / copy_from_user](../1-user-kernel-boundary/copy-from-user-copy-to-user.md): user pointer copy 규칙

## 기억할 문장

driver ioctl을 읽을 때 핵심은 “각 command가 어떤 user data를 받아 어떤 driver state를 바꾸며, 그 전에 모든 field가 검증되는가?”다.
