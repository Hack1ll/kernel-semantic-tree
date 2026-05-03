# current

## 한 문장 정의

`current`는 현재 CPU에서 실행 중인 `task_struct`를 가리키는 커널 기준점이다.

## 왜 중요한가

커널 코드에는 “현재 실행 중인 task”를 기준으로 판단하는 경로가 많다. syscall handler에서 `current`를 보면 그 syscall을 실행 중인 task를 찾을 수 있다.

하지만 `current`는 “원래 요청자”라는 뜻이 아니다. 지금 이 CPU에서 실제로 실행 중인 task라는 뜻이다. worker thread, kernel thread, interrupt context에서는 이 차이가 중요하다.

## syscall에서의 current

일반적인 syscall 경로에서는 user task가 kernel mode로 들어온다. 이때 `current`는 그 user task다.

```text
user task
    -> syscall
    -> kernel handler
    -> current == syscall을 실행 중인 task
```

이 상황에서는 다음 연결이 자연스럽다.

- `current->cred`: syscall을 실행 중인 task의 권한
- `current->files`: syscall을 실행 중인 task의 fd table
- `current->mm`: syscall을 실행 중인 task의 user address space
- `current->nsproxy`: syscall을 실행 중인 task의 namespace 묶음

예를 들어 `read(fd, buf, size)`는 `current->files`에서 fd를 찾고, `buf`는 `current->mm`에 속한 user address space의 주소로 해석된다.

## current가 원래 요청자와 달라지는 경우

비동기 작업에서는 요청을 받은 task와 실제 작업을 수행하는 task가 다를 수 있다.

대표적인 경우는 다음과 같다.

- workqueue worker가 나중에 작업을 실행한다.
- kernel thread가 background cleanup을 수행한다.
- io_uring worker가 요청자의 I/O를 대신 처리한다.
- interrupt handler가 현재 CPU에서 발생한 interrupt를 처리한다.
- timer callback이 원래 요청 이후 별도 시점에 실행된다.

이때 `current`를 권한 판단 기준으로 쓰면 틀릴 수 있다. 원래 요청자의 credential, file, namespace를 따로 저장해 두었는지 확인해야 한다.

## current와 credential

권한 검사는 항상 `current->cred`만 보면 된다고 생각하면 위험하다.

구분해야 할 기준은 다음과 같다.

- 지금 실행 중인 task의 권한: `current->cred`
- file을 open한 시점의 권한: `file->f_cred`
- 특정 namespace 기준 capability: `ns_capable()`
- 임시로 바꾼 credential: `override_creds()` / `revert_creds()`

worker가 작업을 처리하는 경우, 작업을 제출한 task의 권한을 저장해 쓰는지 아니면 worker 자신의 권한을 쓰는지 확인해야 한다.

## current와 user memory

`copy_from_user()`와 `copy_to_user()`는 user address space와 연결된다. 일반 syscall에서는 `current->mm`이 user memory 접근의 기준이 된다.

하지만 kernel thread는 user address space가 없을 수 있다. 따라서 kernel thread나 worker context에서 user pointer를 뒤늦게 사용하려면 그 pointer가 아직 유효한지, 어떤 address space 기준인지 따로 검토해야 한다.

## current와 context switch

context switch가 일어나면 CPU에서 실행 중인 task가 바뀌고, 그 CPU의 `current`도 바뀐다. 다만 함수가 실행 중인 동안 임의로 다른 task의 `current`가 섞여 들어오는 것이 아니다. kernel code가 sleep하거나 schedule 지점에 도달할 수 있는지가 중요하다.

코드에서 확인할 점은 다음과 같다.

1. 이 함수는 syscall 문맥에서 직접 실행되는가?
2. workqueue, timer, interrupt, kernel thread에서 실행될 수 있는가?
3. `current`를 권한 기준으로 써도 원래 요청자와 같은가?
4. user pointer를 저장해 나중에 쓰고 있지 않은가?
5. `override_creds()`를 사용했다면 모든 return path에서 복구하는가?

## 보안 관점

`current` 관련 버그는 “누구의 권한인가”를 잘못 판단할 때 생긴다.

- worker의 `current->cred`를 요청자의 권한으로 착각한다.
- open 시점 권한이 필요한데 현재 task 권한으로 다시 판단한다.
- namespace가 다른 task의 기준으로 capability를 검사한다.
- kernel thread에서 user pointer를 직접 사용한다.
- `override_creds()` 이후 error path에서 `revert_creds()`를 빠뜨린다.

## 다른 문서와의 연결

- [task_struct](task-struct.md): `current`가 가리키는 객체
- [scheduler](scheduler.md): context switch로 `current`가 바뀌는 원리
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syscall과 user pointer
- [7. Permission Model](../7-permission-model/README.md): credential과 capability
- [12. io_uring](../12-io-uring/README.md): worker가 요청을 대신 실행하는 대표 영역

## 기억할 문장

`current`는 “지금 실행 중인 task”이지, 모든 경우에 “원래 요청한 user task”는 아니다.
