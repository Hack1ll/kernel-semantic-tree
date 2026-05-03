# 2. Process / Task

## 핵심 질문

커널은 “지금 실행 중인 주체”를 어떻게 표현하고, 그 주체에 어떤 자원을 연결하는가?

## 큰가지의 의미

리눅스에서 실행 중인 단위는 `task_struct`로 표현된다. process와 thread의 차이를 보기 전에, 커널이 현재 실행 주체를 기준으로 memory, file table, credential, namespace, scheduler state를 따라간다는 점을 잡아야 한다.

```text
task_struct
    -> mm
    -> files
    -> cred
    -> nsproxy
    -> sched state
```

`current`는 단순 편의 macro가 아니다. 많은 kernel path에서 “누가 요청했는가”를 찾는 출발점이다.

## 하위 문서의 역할

- [task_struct](task-struct.md): 실행 주체를 표현하는 중심 object
- [fork / clone / exec / exit](fork-clone-exec-exit.md): task가 생성, 공유, 교체, 종료되는 lifecycle
- [scheduler](scheduler.md): runnable task 중 무엇을 CPU에서 실행할지 결정하는 규칙
- [current](current.md): 현재 CPU에서 실행 중인 task를 찾는 기준
- [task lifetime](task-lifetime.md): task object와 연결 자원이 언제까지 살아 있는지에 대한 규칙

## 이 장에서 특히 구분할 것

process, thread, task는 완전히 같은 말이 아니다. 커널은 실행 단위를 task로 보고, clone flag에 따라 address space, fd table, signal handler 등을 공유하거나 분리한다.

`current`도 항상 “원래 요청자”를 뜻하지 않는다. worker thread, kernel thread, interrupt context에서는 current의 의미가 달라질 수 있다.

## 대표 흐름

```text
open syscall
    -> current task
    -> current->files
    -> fd table update
```

```text
permission check
    -> current->cred
    -> capability / namespace 기준 판단
```

```text
fork
    -> 새 task_struct
    -> mm/files/cred 공유 또는 복사
    -> scheduler에 runnable task 등록
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syscall은 현재 task의 문맥에서 처리된다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd table은 task의 `files_struct`에 붙는다.
- [7. Permission Model](../7-permission-model/README.md): 권한 판단은 task의 `cred`에서 출발한다.
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): task는 namespace 묶음을 통해 다른 view를 본다.

## 보안 관점

task를 잘못 이해하면 요청 주체를 잘못 판단한다.

- worker에서 `current`를 실제 요청자로 착각
- task reference 없이 task pointer 저장
- exec 경계에서 credential 변화를 놓침
- clone flag가 만든 공유 관계를 잘못 가정
- task exit와 procfs, ptrace, signal path의 lifetime race

## 읽고 나서 확인할 것

1. 이 코드에서 현재 실행 주체는 누구인가?
2. `current`가 실제 요청자와 같은가?
3. 이 task가 참조하는 `mm`, `files`, `cred`, `nsproxy` 중 무엇이 핵심인가?
4. task exit 중에도 해당 pointer가 살아 있다고 보장되는가?
