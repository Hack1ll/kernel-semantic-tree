# fd table

## 한 문장 정의

fd table은 process가 가진 작은 정수 fd를 kernel의 `struct file` reference로 바꾸는 task별 handle table이다.

## 왜 중요한가

user space는 kernel pointer를 직접 받지 않는다. 대신 `open()`, `socket()`, `eventfd()`, `bpf()`, `io_uring_setup()` 같은 syscall이 작은 정수 fd를 돌려준다.

커널은 syscall마다 이 숫자를 현재 task의 fd table에서 찾아 실제 open object로 바꾼다.

```text
user fd number
    -> current->files
    -> fd table slot
    -> struct file reference
    -> file operation 실행
```

## fd는 process-local 값이다

fd 숫자는 전역 ID가 아니다. 같은 숫자 `3`이라도 process마다 다른 `struct file`을 가리킬 수 있다.

task는 `task_struct -> files_struct` 연결을 통해 fd table을 가진다. `fork()`나 `clone()` flag에 따라 fd table이 복사되거나 공유될 수 있다.

따라서 fd를 해석할 때는 항상 “어느 task의 fd table인가?”를 먼저 봐야 한다.

## lookup과 reference

fd lookup은 단순 배열 접근이 아니다. lookup 중에 `close()`가 동시에 실행될 수 있으므로, operation을 수행하는 동안 `struct file`이 살아 있도록 reference를 얻어야 한다.

일반 흐름은 다음과 같다.

```text
syscall(fd)
    -> fd table에서 struct file 찾기
    -> file reference 획득
    -> file operation 수행
    -> file reference 반환
```

fd table entry가 제거되어도 이미 reference를 얻은 operation은 `struct file`을 계속 사용할 수 있다.

## close, dup, fork, exec

fd table의 의미는 lifecycle syscall과 함께 바뀐다.

- `close(fd)`: fd table slot을 비우고 `struct file` reference를 put한다.
- `dup(fd)`: 같은 `struct file`을 가리키는 새 fd slot을 만든다.
- `fork()`: child가 parent의 fd table 내용을 물려받을 수 있다.
- `execve()`: `close-on-exec` flag가 설정된 fd는 닫힌다.

`dup()`으로 만들어진 fd들은 같은 `struct file`을 공유하므로 file offset이나 open state를 공유할 수 있다.

## fd reuse

fd 번호는 닫힌 뒤 재사용될 수 있다. user space가 오래 들고 있던 숫자 fd가 나중에 전혀 다른 object를 가리킬 수 있다.

```text
fd 5 -> file A
close(5)
open(file B) -> fd 5 재사용 가능
```

커널 코드가 fd 숫자만 저장해 두고 나중에 다시 lookup하면 다른 object를 얻을 수 있다. 오래 써야 한다면 `struct file` reference를 잡아야 한다.

## fd flag와 file flag

fd table slot에 붙는 flag와 `struct file`에 붙는 flag는 다르다.

- fd flag: fd table entry에 붙는다. 대표적으로 close-on-exec가 있다.
- file flag: open file instance에 붙는다. `O_NONBLOCK`, `O_APPEND` 같은 open 상태와 연결된다.

`dup()`은 같은 `struct file`을 공유하지만 fd flag는 slot마다 다를 수 있다.

## 코드에서 확인할 것

1. 이 fd는 어느 task의 `files_struct`에서 해석되는가?
2. fd lookup 뒤 `struct file` reference를 얻는가?
3. `close()`나 `dup()`과 동시에 실행될 수 있는가?
4. fd 숫자를 저장해 나중에 다시 lookup하지 않는가?
5. fd flag와 file flag를 혼동하지 않는가?
6. `execve()`에서 close-on-exec 처리가 보안상 필요한가?

## 보안 관점

fd table 관련 버그는 handle reuse와 lifetime 착각에서 자주 나온다.

- fd 숫자를 kernel object identity로 오래 저장한다.
- fd lookup 뒤 reference 없이 operation을 진행한다.
- close와 ioctl/read/write가 race를 일으킨다.
- close-on-exec 누락으로 민감한 fd가 새 프로그램에 남는다.
- shared fd table을 독립 table로 착각한다.

## 다른 문서와의 연결

- [struct file](struct-file.md): fd table slot이 가리키는 open instance
- [file_operations](file-operations.md): fd lookup 뒤 실행되는 operation table
- [2. Process / Task](../2-process-task/README.md): `task_struct -> files`
- [4. Object Lifetime](../4-object-lifetime/README.md): file reference와 release
- [7. Permission Model](../7-permission-model/README.md): fd를 통한 권한 유지와 `file->f_cred`

## 기억할 문장

fd table을 읽을 때 핵심은 “이 숫자 fd가 지금 어느 task에서 어떤 `struct file` reference로 바뀌는가?”다.
