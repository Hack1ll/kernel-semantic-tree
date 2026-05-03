# pid ns

## 한 문장 정의

pid namespace는 process id 번호 공간과 보이는 process tree를 분리해, 같은 task가 namespace마다 다른 pid로 보일 수 있게 한다.

## 왜 중요한가

container 안에서 보이는 PID 1은 host에서 다른 pid를 가진 task일 수 있다. pid namespace는 process를 복제하지 않는다. 같은 `task_struct`를 namespace별 pid number로 다르게 보여준다.

```text
task_struct
    -> struct pid
    -> pid number in host pid ns
    -> pid number in container pid ns
```

따라서 pid 숫자만으로 task identity를 판단하면 안 된다.

## nested pid namespace

pid namespace는 중첩될 수 있다. child namespace의 task는 parent namespace에서도 보일 수 있지만, parent namespace의 모든 task가 child namespace 안에서 보이는 것은 아니다.

```text
host pid ns
    -> pid 12345
container pid ns
    -> same task appears as pid 1
```

커널은 `struct pid`를 통해 여러 namespace의 pid number를 함께 관리한다.

## PID 1 의미

각 pid namespace에는 PID 1이 있다. 이 task는 그 namespace 내부에서 init 역할을 한다.

중요한 특징은 다음이다.

- namespace 내부 orphan process를 reparent할 수 있다.
- signal 처리 의미가 일반 process와 다를 수 있다.
- PID 1이 종료되면 namespace 안의 process들도 영향을 받을 수 있다.
- container lifecycle과 연결된다.

container 내부 PID 1과 host init process를 혼동하면 안 된다.

## procfs와 visibility

`/proc`은 pid namespace와 강하게 연결된다. container 내부 `/proc`이 올바른 pid namespace 기준으로 mount되어야 host process 정보가 과도하게 보이지 않는다.

확인할 점은 다음이다.

- `/proc` mount가 container pid namespace 기준인가?
- host pid가 container 내부에 노출되는가?
- procfs entry가 task lifetime을 안전하게 붙잡는가?
- pid namespace cleanup 중 procfs lookup이 race나지 않는가?

## signal, ptrace, wait

pid namespace는 process visibility와 권한 check에 영향을 준다.

- signal target pid가 어느 namespace 기준으로 해석되는가?
- ptrace가 parent/child namespace 관계를 고려하는가?
- wait 대상 child가 같은 pid namespace 관계 안에 있는가?
- pidfd가 namespace 경계를 어떻게 다루는가?

task가 보인다는 것과 조작 권한이 있다는 것은 별개다. user namespace와 credential check가 함께 필요하다.

## 코드에서 확인할 것

1. pid 숫자는 어느 pid namespace 기준인가?
2. `struct pid`와 `task_struct` 중 무엇을 저장하고 있는가?
3. pid lookup이 caller namespace 기준인지 target namespace 기준인지 명확한가?
4. `/proc` mount가 올바른 pid namespace view를 보여주는가?
5. signal, ptrace, wait 권한 check가 namespace 관계를 고려하는가?
6. pid namespace exit 중 task reference와 procfs entry가 안전하게 정리되는가?

## 보안 관점

pid namespace 버그는 process 정보 노출이나 조작 권한 우회로 이어질 수 있다.

- host pid를 container 내부에 그대로 노출한다.
- pid 숫자만 저장해 pid reuse나 namespace 차이를 놓친다.
- procfs가 host process 정보를 보여준다.
- signal/ptrace target을 잘못된 namespace 기준으로 해석한다.
- namespace cleanup 중 task pointer lifetime을 놓쳐 UAF가 생긴다.

## 다른 문서와의 연결

- [2. Process / Task](../2-process-task/README.md): `task_struct`, `struct pid`, task lifetime
- [user ns](user-ns.md): process 조작 권한 기준
- [mount ns](mount-ns.md): `/proc` mount view
- [7. Permission Model](../7-permission-model/README.md): signal, ptrace permission
- [4. Object Lifetime](../4-object-lifetime/README.md): task and pid reference

## 기억할 문장

pid namespace를 읽을 때 핵심은 “이 pid 숫자가 어느 namespace에서 어떤 `task_struct`를 가리키는가?”다.
