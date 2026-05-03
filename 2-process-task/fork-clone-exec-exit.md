# fork / clone / exec / exit

## 한 문장 정의

`fork`, `clone`, `exec`, `exit`는 task가 만들어지고, 무엇을 공유할지 정하고, 실행 이미지를 교체하고, 종료되는 경로다.

## 왜 중요한가

process와 thread의 차이는 커널 안에서 “새 task가 어떤 객체를 공유하는가”로 나타난다. `fork()`는 새 실행 주체를 만들고, `clone()`은 공유 범위를 세밀하게 정하며, `execve()`는 같은 task 위에서 프로그램 이미지를 바꾸고, `exit()`는 task가 잡고 있던 자원을 내려놓는다.

이 문서는 task의 “구조”보다 “변화”에 집중한다.

```text
fork / clone
    -> 새 task_struct 생성
    -> mm, files, signal, namespace 공유 또는 복사
    -> scheduler에 runnable task로 등록

execve
    -> 같은 task 유지
    -> 주소 공간과 실행 이미지 교체
    -> 필요하면 credential 갱신

exit
    -> 실행 중단
    -> mm, files, signal 같은 연결 자원 정리
    -> parent가 wait할 종료 상태 보관
```

## fork

`fork()`는 호출한 task를 기준으로 새 task를 만든다. 새 task는 부모의 상태를 많이 물려받지만, 모든 것을 그대로 공유하는 것은 아니다.

중요한 변화는 다음과 같다.

- 새 `task_struct`가 생긴다.
- address space는 copy-on-write 방식으로 분리될 수 있다.
- fd table은 복사되지만, fd가 가리키는 `struct file`은 reference를 공유할 수 있다.
- credential과 namespace는 reference를 통해 이어질 수 있다.
- 새 task는 scheduler가 실행할 수 있는 상태가 된다.

`fork()`를 볼 때는 “복사된 것”과 “공유된 것”을 구분해야 한다.

## clone

`clone()`은 task 생성 시 공유 범위를 flag로 지정한다. 이 flag 조합이 process와 thread의 차이를 만든다.

대표적으로 구분할 항목은 다음과 같다.

- 주소 공간을 공유하는가?
- fd table을 공유하는가?
- signal handler를 공유하는가?
- thread group에 들어가는가?
- namespace를 새로 만들거나 공유하는가?

예를 들어 thread 생성은 여러 객체를 공유하는 `clone()`으로 이해할 수 있다. 반대로 namespace 생성은 task는 새로 만들면서 일부 view를 분리하는 경로로 연결된다.

## exec

`execve()`는 새 task를 만드는 syscall이 아니다. 기존 task가 유지된 상태에서 실행할 프로그램 이미지와 주소 공간이 교체된다.

여기서 주의할 점은 다음과 같다.

- pid는 유지된다.
- fd table은 기본적으로 유지되지만 `close-on-exec` fd는 닫힌다.
- 기존 user address space는 새 프로그램의 address space로 바뀐다.
- setuid, file capability 같은 조건에 따라 credential이 바뀔 수 있다.
- signal handler, thread 관계, memory mapping 상태가 재정리된다.

그래서 `execve()` 경계는 권한과 메모리 의미가 크게 바뀌는 지점이다.

## exit

`exit()`는 task가 실행을 끝내는 경로다. 하지만 “task가 exit했다”와 “`task_struct` 메모리가 바로 사라졌다”는 같은 말이 아니다.

종료 경로에서는 보통 다음 일이 일어난다.

- user address space를 내려놓는다.
- 열린 file reference를 정리한다.
- signal, futex, ptrace, cgroup 관련 상태를 정리한다.
- parent가 `wait()`로 읽을 수 있는 종료 상태를 남긴다.
- 마지막 reference가 사라질 때 실제 task object가 해제된다.

상세한 해제 규칙은 [task lifetime](task-lifetime.md)에서 따로 본다.

## 한 흐름으로 보기

shell에서 명령 하나를 실행하면 다음 순서를 자주 만난다.

```text
shell task
    -> fork로 child task 생성
    -> child가 execve로 새 프로그램 이미지 로드
    -> 프로그램 실행
    -> exit으로 종료
    -> shell이 wait으로 종료 상태 회수
```

이 흐름에서 child task는 `fork()` 뒤 새로 생기지만, `execve()` 뒤에도 같은 task다.

## 코드에서 확인할 것

1. 새 task가 만들어지는가, 기존 task가 바뀌는가?
2. `mm`, `files`, `cred`, `nsproxy`, `sighand` 중 무엇이 공유되는가?
3. 공유 객체의 reference count가 증가하는가?
4. 실패 경로에서 이미 잡은 reference를 되돌리는가?
5. `execve()` 이후에도 유지된다고 가정한 fd나 credential이 정말 유지되는가?
6. `exit()` 중인 task를 다른 경로가 동시에 참조할 수 있는가?

## 보안 관점

이 영역의 취약점은 생성과 정리 경계에서 많이 보인다.

- `clone()` flag 조합을 잘못 해석해 namespace나 credential 기준을 틀린다.
- `execve()` 중 credential 변경 전후의 검사를 섞는다.
- `close-on-exec` 처리 누락으로 민감한 fd가 새 프로그램에 남는다.
- `fork()` 실패 경로에서 reference를 덜 해제하거나 두 번 해제한다.
- `exit()`와 ptrace, procfs, signal 경로가 같은 task를 동시에 만진다.

## 다른 문서와의 연결

- [task_struct](task-struct.md): 생성되는 중심 객체
- [scheduler](scheduler.md): 새 task가 runnable 상태로 들어가는 경로
- [task lifetime](task-lifetime.md): exit 이후 object가 실제로 사라지는 조건
- [7. Permission Model](../7-permission-model/README.md): `execve()`와 credential 변화
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): `clone()`과 namespace 생성

## 기억할 문장

`fork`와 `clone`은 task를 만들고, `exec`는 같은 task의 실행 내용을 바꾸며, `exit`는 실행을 끝내지만 task object의 해제는 reference 규칙을 따른다.
