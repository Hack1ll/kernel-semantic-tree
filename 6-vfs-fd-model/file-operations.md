# file_operations

## 한 문장 정의

`file_operations`는 VFS가 `read`, `write`, `ioctl`, `mmap`, `poll`, `release` 같은 file operation을 실제 filesystem, driver, socket, subsystem 구현으로 dispatch하기 위한 함수 table이다.

## 왜 중요한가

리눅스에서 fd는 regular file만 가리키지 않는다. device, socket, pipe, eventfd, timerfd, bpf object, io_uring도 fd로 노출될 수 있다.

VFS는 fd lookup으로 얻은 `struct file`의 `f_op`를 보고 실제 함수를 호출한다.

```text
read(fd, buf, len)
    -> fd table lookup
    -> struct file
    -> file->f_op->read_iter 또는 read
    -> filesystem / driver / socket 구현
```

## operation table

`file_operations`에는 open instance에 대해 수행할 수 있는 callback이 들어간다.

대표 callback은 다음과 같다.

- `.open`: open 시점 초기화
- `.release`: 마지막 close 시점 정리
- `.read` / `.read_iter`: read 처리
- `.write` / `.write_iter`: write 처리
- `.unlocked_ioctl`: ioctl command 처리
- `.compat_ioctl`: 32-bit compat ioctl 처리
- `.mmap`: user address space mapping 처리
- `.poll`: poll/select/epoll readiness 처리
- `.llseek`: file offset 변경

구현마다 없는 operation도 있다. 없는 operation을 호출할 때 VFS가 어떤 error를 반환하는지도 봐야 한다.

## dispatch 흐름

VFS는 syscall 이름만 보고 모든 구현을 알지 않는다. `struct file`의 operation table이 실제 코드를 결정한다.

```text
syscall
    -> fd lookup
    -> struct file reference
    -> f_op callback 선택
    -> callback에 file pointer 전달
    -> callback이 private object 조작
```

그래서 같은 `read(fd)`라도 regular file, socket, character device에서 완전히 다른 code path로 갈 수 있다.

## open과 release

`.open`은 file이 만들어질 때 subsystem-specific state를 붙이는 지점이다. `.release`는 마지막 file reference가 사라질 때 정리하는 지점이다.

특히 driver code에서는 다음 패턴이 자주 보인다.

```text
open
    -> device object lookup
    -> reference 획득
    -> file->private_data 설정

release
    -> file->private_data 회수
    -> reference 반환
```

open 실패 경로와 release 경로가 같은 resource를 두 번 정리하지 않는지 확인해야 한다.

## ioctl과 mmap

`ioctl`과 `mmap`은 file operation 중에서도 취약점 연구에서 자주 보는 entry point다.

- `ioctl`: command별 user input 구조체를 해석하고 subsystem state를 바꾼다.
- `mmap`: file이나 device buffer를 user address space에 mapping한다.

이 두 callback은 user pointer, size, offset, permission, object lifetime이 한 번에 만나는 경우가 많다.

## file_operations와 private_data

operation callback은 보통 `struct file *file`을 인자로 받는다. 실제 object는 `file->private_data`, inode, socket wrapper, filesystem-specific state를 통해 찾아간다.

따라서 callback을 읽을 때는 다음을 봐야 한다.

- `private_data`는 어디서 설정되었는가?
- 어떤 타입으로 cast되는가?
- release 전까지 살아 있는가?
- concurrent operation이 같은 private object를 만지는가?

## 코드에서 확인할 것

1. 이 fd의 `file->f_op`는 어느 subsystem의 table인가?
2. callback이 `file->private_data`를 어떤 타입으로 해석하는가?
3. `.open`에서 얻은 reference가 `.release`에서 반환되는가?
4. `.ioctl`과 `.mmap`에서 user input 검증이 command별로 충분한가?
5. `.release`와 in-flight operation이 race나지 않는가?
6. compat path가 native path와 같은 검증을 하는가?

## 보안 관점

`file_operations` 주변 취약점은 callback dispatch 이후의 타입과 lifetime에서 자주 나온다.

- `private_data` type confusion
- ioctl command별 size 검증 누락
- compat ioctl 구조체 해석 오류
- mmap offset/length overflow
- release와 read/write/ioctl 동시 실행으로 UAF
- `.poll`이나 async notification callback이 해제된 object를 참조

## 다른 문서와의 연결

- [fd table](fd-table.md): fd에서 `struct file`을 얻는 과정
- [struct file](struct-file.md): `f_op`와 `private_data`가 붙는 open instance
- [mmap](../3-memory-management/mmap.md): file operation으로서의 mmap
- [ioctl](../10-device-drivers/ioctl.md): driver ioctl entry point
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): user input이 operation callback에 들어오는 경계

## 기억할 문장

`file_operations`를 읽을 때 핵심은 “이 fd의 operation이 실제로 어느 callback으로 dispatch되고, 그 callback이 어떤 private object를 만지는가?”다.
