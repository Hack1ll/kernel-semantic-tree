# 6. VFS / FD Model

## 핵심 질문

리눅스는 왜 파일, socket, pipe, device, BPF object, io_uring까지 fd로 다루는가?

## 큰가지의 의미

fd는 단순한 숫자가 아니다. user space가 kernel object를 직접 포인터로 받지 않도록 하는 handle이고, 동시에 open된 object에 대한 reference다.

VFS는 다양한 object를 file operation이라는 공통 모양으로 다루게 한다.

```text
path or syscall
    -> struct file
    -> file_operations
    -> filesystem / socket / driver / subsystem
```

## 하위 문서의 역할

- [fd table](fd-table.md): fd 번호를 `struct file` reference로 바꾸는 table
- [struct file](struct-file.md): 열린 file instance와 open state를 표현하는 object
- [inode / dentry](inode-dentry.md): filesystem object와 path name cache
- [file_operations](file-operations.md): VFS operation을 실제 구현 함수로 연결하는 table
- [path lookup](path-lookup.md): 문자열 경로를 dentry, inode, mount로 해석하는 과정
- [mount namespace](mount-namespace.md): path view를 namespace별로 분리하는 규칙

## 이 장에서 특히 구분할 것

path와 fd는 다르다.  
inode와 `struct file`도 다르다.  
open 시점 권한과 사용 시점 권한도 다를 수 있다.

```text
path string
    -> dentry / inode
    -> open
    -> struct file
    -> fd table slot
    -> file operation
```

## 대표 흐름

```text
open("/dev/foo")
    -> path lookup
    -> inode permission
    -> struct file 생성
    -> file_operations 연결
    -> fd 반환
```

```text
ioctl(fd, cmd, arg)
    -> fd table lookup
    -> struct file
    -> file->f_op->unlocked_ioctl
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): fd는 syscall boundary에서 kernel object로 변환된다.
- [2. Process / Task](../2-process-task/README.md): fd table은 task의 `files_struct`에 연결된다.
- [7. Permission Model](../7-permission-model/README.md): `file->f_cred`와 current credential의 차이가 중요하다.
- [10. Device Drivers](../10-device-drivers/README.md): driver는 device를 file operation으로 노출한다.

## 보안 관점

VFS/FD bug는 handle과 object lifetime의 착각에서 자주 나온다.

- fd reuse
- `file->private_data` type confusion
- close와 operation race
- path lookup TOCTOU
- mount namespace 기준 오류
- open 시점 권한과 use 시점 권한 혼동

## 읽고 나서 확인할 것

1. user가 넘긴 값은 path인가 fd인가?
2. fd가 어떤 `struct file`로 변환되는가?
3. `struct file` 뒤의 실제 subsystem object는 무엇인가?
4. close, dup, fork, exec가 이 fd에 어떤 영향을 주는가?
5. path 기준 권한과 open file 기준 권한이 섞이지 않았는가?
