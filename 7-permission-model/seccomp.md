# seccomp

## 한 문장 정의

seccomp는 process가 kernel에 진입할 때 사용할 수 있는 syscall과 일부 syscall argument를 filter로 제한하는 sandboxing 메커니즘이다.

## 왜 중요한가

capability와 LSM은 주로 object와 operation 권한을 판단한다. seccomp는 그보다 앞선 syscall entry 단계에서 “이 syscall을 호출해도 되는가?”를 제한한다.

```text
user process
    -> syscall 진입
    -> seccomp filter 평가
    -> allow / errno / kill / trap / notify
    -> 허용 시 syscall handler 실행
```

seccomp는 공격면을 줄이는 데 강하지만, object의 실제 의미를 깊게 이해하는 권한 모델은 아니다.

## filter가 보는 정보

seccomp filter는 syscall metadata를 본다.

- architecture
- syscall number
- syscall arguments 값
- instruction pointer 등 제한된 context

하지만 pointer argument가 가리키는 user memory 내용을 일반적으로 안전하게 깊게 검사하지 않는다. 예를 들어 `openat()`의 path 문자열 내용을 seccomp filter가 일반 permission check처럼 해석하는 구조가 아니다.

## action

filter 결과는 action으로 표현된다.

- allow: syscall 실행 허용
- errno: syscall을 실행하지 않고 지정 error 반환
- kill: task 또는 process 종료
- trap: signal 기반 처리
- trace: tracer가 관찰하거나 개입
- user notification: supervisor process가 결정을 돕는 구조
- log: 정책에 따라 logging

정책을 읽을 때는 deny 방식이 무엇인지 봐야 한다. 같은 차단이라도 errno 반환과 kill은 프로그램 동작에 미치는 영향이 다르다.

## no_new_privs와 설치 조건

seccomp filter를 설치할 때 `no_new_privs`가 중요한 조건으로 등장한다. unprivileged process가 filter를 설치해도 이후 exec에서 더 높은 privilege를 얻어 filter를 우회하지 못하게 하려는 목적이다.

권한 있는 process는 다른 조건으로 filter를 설치할 수 있지만, sandbox 설계에서는 `no_new_privs`와 exec transition을 함께 봐야 한다.

## thread와 filter inheritance

seccomp는 process와 thread 구조에서 전파 규칙이 중요하다.

- filter는 추가될수록 완화되지 않고 보통 더 제한적이 된다.
- fork/clone/exec 이후 filter가 유지될 수 있다.
- thread sync 옵션을 쓰면 thread group 전체에 filter를 맞추려 한다.
- 이미 실행 중인 thread가 다른 filter 상태를 갖는지 확인해야 한다.

multi-threaded program에서 filter 적용 범위가 틀리면 sandbox가 부분적으로만 적용될 수 있다.

## seccomp의 한계

seccomp는 syscall surface를 줄이는 도구다. 하지만 syscall이 허용된 뒤의 object permission은 여전히 별도 check가 필요하다.

예를 들어 `ioctl` 하나를 허용하면 그 안의 command space는 매우 넓을 수 있다. `openat`을 허용하면 path namespace, fd, mount, LSM 정책이 다시 중요해진다.

```text
seccomp allow ioctl
    -> ioctl syscall 진입 가능
    -> fd lookup
    -> file_operations dispatch
    -> command별 permission과 input 검증 필요
```

## user notification

seccomp user notification은 supervisor가 syscall 결정을 도울 수 있게 한다. 이 기능은 강력하지만 race와 TOCTOU를 조심해야 한다.

확인할 점은 다음과 같다.

- supervisor가 본 pid/fd/path 정보가 결정 시점까지 같은 의미인가?
- target task가 다른 thread에서 상태를 바꿀 수 있는가?
- fd가 재사용될 수 있는가?
- path 문자열을 다시 lookup해 다른 object를 보지 않는가?

## 코드에서 확인할 것

1. filter가 syscall number와 architecture를 모두 확인하는가?
2. argument 값을 검사할 때 pointer가 가리키는 내용을 과신하지 않는가?
3. deny action이 프로그램에 맞게 선택되었는가?
4. filter가 fork/clone/exec/thread에 어떻게 전파되는가?
5. 허용된 syscall 내부 command space가 너무 넓지 않은가?
6. user notification 사용 시 fd/path/pid race를 고려하는가?

## 보안 관점

seccomp 관련 문제는 sandbox 우회나 공격면 과다 노출로 나타난다.

- architecture check 누락으로 다른 syscall numbering을 허용한다.
- `ioctl`, `bpf`, `clone`, `unshare`, `mount` 같은 넓은 syscall을 과도하게 허용한다.
- pointer argument 내용을 filter가 검증했다고 착각한다.
- multi-threaded process 일부에만 filter가 적용된다.
- user notification supervisor가 TOCTOU에 취약하다.
- seccomp만 믿고 capability, LSM, namespace check를 생략한다.

## 다른 문서와의 연결

- [cred](cred.md): seccomp filter를 설치하고 유지하는 task의 credential
- [capabilities](capabilities.md): filter 설치 조건과 privileged operation
- [LSM](lsm.md): syscall entry 제한과 object policy의 차이
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syscall entry point
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): 허용된 fd 기반 syscall 내부 동작

## 기억할 문장

seccomp를 읽을 때 핵심은 “이 sandbox가 syscall entry를 얼마나 줄이고, 허용된 syscall 내부의 object 권한은 어디서 다시 검증되는가?”다.
