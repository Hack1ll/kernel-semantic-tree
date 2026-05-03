# task_struct

## 한 문장 정의

`task_struct`는 리눅스가 실행 중인 task를 표현하고, 그 task가 사용하는 주소 공간, 파일 테이블, 권한, namespace, scheduler 상태로 이어지는 중심 구조체다.

## 왜 중요한가

커널 코드를 읽을 때 “이 요청은 누구의 권한으로 실행되는가?”, “어떤 fd table을 쓰는가?”, “어떤 주소 공간을 대상으로 하는가?”를 계속 만나게 된다. 이 질문의 출발점이 대부분 `task_struct`다.

`task_struct` 자체가 모든 정보를 직접 들고 있는 것은 아니다. 중요한 점은 여러 커널 객체를 가리키는 연결점이라는 것이다.

```text
task_struct
    -> mm_struct       user address space
    -> files_struct    fd table
    -> cred            uid, gid, capabilities
    -> nsproxy         namespace 묶음
    -> signal_struct   process group, signal state
    -> sched_entity    scheduler가 보는 실행 상태
```

따라서 `task_struct`를 볼 때는 필드를 하나씩 외우기보다 “이 필드가 어느 큰가지로 연결되는가”를 먼저 잡아야 한다.

## 커널이 task_struct를 쓰는 방식

```text
user program
    -> syscall 진입
    -> current task_struct 확인
    -> current->files 또는 current->cred 같은 연결 객체 사용
    -> 요청한 kernel object 상태 변경
```

예를 들어 `open()` 계열 syscall은 현재 task의 권한을 보고, 성공하면 현재 task의 fd table에 새 fd를 넣는다. `mmap()`은 현재 task의 `mm_struct`에 새 virtual memory area를 붙인다.

## 주요 연결

### `task_struct -> mm`

`mm`은 user address space를 가리킨다. user pointer 검증, `mmap`, page fault, `execve` 이후 주소 공간 교체를 이해할 때 필요하다.

### `task_struct -> files`

`files`는 fd table을 가리킨다. `open`, `close`, `dup`, `ioctl`, `read`, `write`가 어떤 file object를 대상으로 하는지 여기서 출발한다.

### `task_struct -> cred`

`cred`는 현재 task의 권한 상태다. uid, gid, capability, user namespace와 연결된다. 권한 검사가 `current->cred` 기준인지, open 시점의 `file->f_cred` 기준인지 구분해야 한다.

### `task_struct -> nsproxy`

`nsproxy`는 mount, pid, network, ipc 같은 namespace 묶음으로 이어진다. 같은 syscall도 task가 속한 namespace에 따라 보이는 객체와 권한 기준이 달라진다.

### `task_struct -> scheduler state`

scheduler는 task가 runnable인지, sleeping인지, 어느 CPU runqueue에 있는지를 관리한다. 이 정보는 CPU를 얻는 문제와 연결된다.

## process, thread, task 구분

리눅스 커널 내부의 실행 단위는 task다. 사용자가 말하는 process와 thread는 `clone()` flag가 만든 공유 관계로 구분된다.

- process에 가까운 task: 주소 공간, fd table, signal handler를 많이 분리한다.
- thread에 가까운 task: `mm`, `files`, `sighand` 등을 다른 task와 공유할 수 있다.
- 커널 관점의 공통점: 둘 다 `task_struct`로 표현된다.

이 구분을 놓치면 “process마다 하나의 fd table이 있다” 같은 단순화가 깨지는 지점에서 코드를 잘못 읽게 된다.

## 코드에서 확인할 것

1. 이 경로가 기준으로 삼는 task는 `current`인가, 인자로 받은 다른 task인가?
2. 필요한 정보가 `task_struct` 안에 직접 있는가, 아니면 연결된 객체에 있는가?
3. `mm`, `files`, `cred`, `nsproxy` 중 어떤 객체의 lifetime이 핵심인가?
4. 이 task pointer를 저장한다면 reference를 잡고 있는가?
5. task가 exit 중이어도 이 필드에 접근해도 되는가?

## 보안 관점

`task_struct` 관련 실수는 보통 “요청 주체를 잘못 잡는 문제”로 이어진다.

- worker thread의 `current`를 원래 요청자로 착각한다.
- task pointer를 reference 없이 저장해 exit 이후 접근한다.
- `current->cred`와 `file->f_cred`의 의미 차이를 놓친다.
- namespace 기준을 task에서 따라가야 하는데 global 상태로 판단한다.
- thread가 공유하는 객체를 독립 객체라고 가정한다.

## 다른 문서와의 연결

- [current](current.md): 현재 CPU에서 실행 중인 task를 찾는 기준
- [fork / clone / exec / exit](fork-clone-exec-exit.md): task가 만들어지고 바뀌고 끝나는 경로
- [task lifetime](task-lifetime.md): task pointer와 연결 객체가 언제까지 유효한지
- [7. Permission Model](../7-permission-model/README.md): `cred`, capability, user namespace
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): `files_struct`, fd table, `struct file`

## 기억할 문장

`task_struct`는 실행 주체 그 자체라기보다, 실행 주체가 가진 권한과 자원으로 들어가는 중심 연결점이다.
