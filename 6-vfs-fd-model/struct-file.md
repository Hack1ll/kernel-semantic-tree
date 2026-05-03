# struct file

## 한 문장 정의

`struct file`은 path를 open한 결과로 만들어지는 open file instance이며, fd가 실제 operation을 수행할 때 도달하는 VFS object다.

## 왜 중요한가

`struct file`은 filesystem inode 자체가 아니다. fd 숫자도 아니다. “열린 상태”를 담는 object다.

같은 path를 두 번 open하면 서로 다른 `struct file`이 생길 수 있다. 반대로 `dup()`이나 `fork()`로 여러 fd가 같은 `struct file`을 공유할 수도 있다.

```text
path
    -> open
    -> struct file
    -> fd table slot
    -> read / write / ioctl / mmap
```

## struct file이 담는 것

`struct file`은 open instance의 상태를 담는다.

대표적으로 다음 정보를 따라간다.

- `f_path`: dentry와 mount를 포함한 path
- `f_inode`: 대상 inode로 이어지는 helper 경로
- `f_op`: 실제 operation 함수 table
- `f_flags`: open flag
- `f_mode`: read/write 가능 mode
- `f_pos`: file offset
- `f_cred`: open 시점 credential
- `private_data`: filesystem, driver, subsystem이 붙이는 private object

필드마다 lifetime과 lock 규칙이 다르므로 “`struct file`이 있으니 모든 뒤쪽 object가 안전하다”고 보면 안 된다.

## open instance와 inode의 차이

inode는 filesystem object의 identity와 metadata를 표현한다. `struct file`은 그 inode를 open한 하나의 사용 상태를 표현한다.

```text
same inode
    -> open #1 -> struct file A -> f_pos A
    -> open #2 -> struct file B -> f_pos B

dup(file A)
    -> fd 3 and fd 4 -> same struct file A -> shared f_pos
```

이 차이를 놓치면 file offset, permission, private data lifetime을 잘못 해석하게 된다.

## f_cred

`file->f_cred`는 file이 open될 때의 credential을 보존한다. 이후 syscall을 실행하는 task의 `current->cred`와 다를 수 있다.

확인해야 할 질문은 다음이다.

- operation이 open 시점 권한을 기준으로 해야 하는가?
- 지금 호출한 task의 권한을 다시 봐야 하는가?
- worker thread가 operation을 수행하면 어떤 credential을 쓰는가?
- namespace 기준 capability check가 필요한가?

권한 모델과 VFS lifetime이 만나는 지점이다.

## private_data

`file->private_data`는 filesystem, driver, socket, subsystem이 open instance에 붙이는 private pointer다.

예를 들어 character device는 open에서 device-specific object를 `private_data`에 넣고, read/write/ioctl에서 다시 꺼내 쓸 수 있다.

위험한 지점은 다음과 같다.

- `private_data` 타입을 잘못 가정한다.
- open 실패 경로에서 초기화된 `private_data`를 정리하지 않는다.
- release 전에 다른 callback이 `private_data`를 계속 참조한다.
- fd가 다른 object로 재사용되었는데 같은 타입으로 착각한다.

## release

`close(fd)`는 fd table entry를 제거하지만, `struct file`이 즉시 release되는 것은 아닐 수 있다. 다른 fd, in-flight operation, kernel reference가 남아 있으면 마지막 reference까지 기다린다.

마지막 file reference가 사라질 때 `f_op->release` 같은 정리 경로가 실행된다.

## 코드에서 확인할 것

1. 이 `struct file`은 path open으로 만들어졌는가, socket/device/subsystem fd인가?
2. 여러 fd가 같은 `struct file`을 공유할 수 있는가?
3. `file->f_op`는 어떤 operation table을 가리키는가?
4. `file->private_data`의 타입과 lifetime이 명확한가?
5. 권한 기준이 `current->cred`인가, `file->f_cred`인가?
6. `release`가 실행될 때 아직 callback이나 mmap이 private object를 참조하지 않는가?

## 보안 관점

`struct file` 버그는 open state와 실제 backing object를 혼동할 때 생긴다.

- `private_data` type confusion
- open 시점 권한과 use 시점 권한 혼동
- close 이후에도 in-flight operation이 남는 상황을 고려하지 않음
- shared `struct file`의 file offset을 독립적이라고 가정
- release path에서 내부 object를 너무 빨리 free

## 다른 문서와의 연결

- [fd table](fd-table.md): fd 숫자가 `struct file` reference로 바뀌는 과정
- [inode / dentry](inode-dentry.md): `struct file`이 연결되는 filesystem object
- [file_operations](file-operations.md): `struct file`을 통해 dispatch되는 operation
- [7. Permission Model](../7-permission-model/README.md): `file->f_cred`와 current credential
- [10. Device Drivers](../10-device-drivers/README.md): driver의 `private_data`와 release

## 기억할 문장

`struct file`을 읽을 때 핵심은 “이 fd가 가리키는 열린 인스턴스의 상태와 그 뒤의 실제 object가 무엇인가?”다.
