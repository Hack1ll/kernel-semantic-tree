# task lifetime

## 한 문장 정의

task lifetime은 task가 생성된 뒤 실행을 끝내고, zombie 상태를 거쳐, 마지막 reference가 사라질 때까지 어떤 상태와 참조 규칙을 따르는지 설명한다.

## 왜 중요한가

task는 exit하는 순간 바로 사라지지 않는다. 실행은 끝났지만 parent가 종료 상태를 회수해야 할 수 있고, procfs, ptrace, signal, pid lookup 같은 경로가 task를 아직 볼 수 있다.

따라서 task lifetime에서 핵심은 “실행 중인가?”보다 “이 pointer를 지금 사용해도 되는가?”다.

```text
task 생성
    -> pid 연결
    -> runnable / running
    -> exit 시작
    -> mm, files 등 연결 자원 정리
    -> zombie 상태로 종료 정보 유지
    -> wait으로 parent가 회수
    -> 마지막 reference 해제
    -> task_struct free
```

## task가 살아 있다는 말의 여러 의미

커널 코드에서 “task가 살아 있다”는 말은 문맥에 따라 다르게 해석된다.

- CPU에서 실행될 수 있다.
- pid lookup으로 찾을 수 있다.
- parent가 wait할 종료 상태가 남아 있다.
- `task_struct` 메모리가 아직 해제되지 않았다.
- RCU reader가 안전하게 볼 수 있는 기간 안에 있다.

이 의미들을 섞으면 exit race나 use-after-free를 놓치기 쉽다.

## reference 규칙

task pointer를 잠깐 보는 것과 저장해 두고 나중에 쓰는 것은 다르다.

나중에 접근하려면 보통 reference가 필요하다.

- `get_task_struct()`: task object reference를 잡는다.
- `put_task_struct()`: 잡은 reference를 놓는다.
- `struct pid`: 숫자 pid 재사용 문제를 피하기 위해 pid 객체를 통해 task를 찾는 데 쓰인다.
- RCU: pid lookup이나 task list 순회에서 reader가 안전하게 보는 기간을 제공한다.

핵심은 pointer 값 자체가 남아 있어도 object lifetime이 보장되지 않으면 접근하면 안 된다는 점이다.

## exit 경로에서 정리되는 것

task가 exit하면 연결된 자원이 단계적으로 정리된다.

- `mm`: user address space와 memory mapping을 내려놓는다.
- `files`: 열린 file reference를 정리한다.
- `fs`: current working directory, root 같은 filesystem context를 정리한다.
- signal 관련 상태: parent 통지, thread group 정리, pending signal 처리와 연결된다.
- cgroup, namespace, audit, ptrace 관련 상태도 각 subsystem 규칙에 따라 정리된다.

이 정리는 한 번에 끝나는 단순 free가 아니다. 여러 subsystem cleanup이 순서대로 실행되고, 일부 상태는 wait 전까지 남을 수 있다.

## zombie와 wait

zombie는 실행은 끝났지만 종료 상태가 남아 있는 task다. parent가 `wait()` 계열 syscall로 종료 상태를 회수하면 zombie로 남아 있던 정보가 정리될 수 있다.

여기서 중요한 점은 다음과 같다.

- zombie task는 user code를 더 실행하지 않는다.
- 종료 code와 signal 정보는 parent가 읽을 수 있게 남는다.
- parent가 없거나 무시하면 reparenting이나 init task가 관련된다.
- zombie가 보인다고 해서 task의 모든 연결 자원이 살아 있는 것은 아니다.

## pid lifetime과 pid reuse

숫자 pid는 재사용될 수 있다. 그래서 오래 보관한 정수 pid만으로 “같은 task”라고 판단하면 안 된다.

커널은 `struct pid`를 통해 pid namespace별 pid와 task 연결을 관리한다. pid를 오래 참조해야 한다면 정수 pid보다 pid object와 reference 규칙을 봐야 한다.

```text
numeric pid
    -> 같은 숫자가 나중에 다른 task에 재사용될 수 있음

struct pid
    -> namespace별 pid 의미와 reference를 함께 관리
```

## 코드에서 확인할 것

1. task pointer를 어디서 얻었는가?
2. lookup 뒤 바로 쓰는가, 저장해 두고 나중에 쓰는가?
3. 저장한다면 `get_task_struct()` 또는 pid reference가 있는가?
4. exit 중인 task의 어떤 필드를 읽고 있는가?
5. RCU read-side critical section 안에서만 유효한 pointer를 밖으로 가져오지 않는가?
6. 숫자 pid를 task identity로 오래 보관하지 않는가?

## 보안 관점

task lifetime 문제는 use-after-free와 race로 이어질 수 있다.

- task pointer를 reference 없이 callback에 저장한다.
- procfs entry가 task exit 이후 stale pointer를 사용한다.
- ptrace나 signal 경로가 exit 중인 task 상태를 잘못 읽는다.
- pid 재사용 때문에 다른 task를 같은 task로 착각한다.
- RCU로 찾은 pointer를 RCU 보호 범위 밖에서 사용한다.

## 다른 문서와의 연결

- [task_struct](task-struct.md): lifetime이 적용되는 중심 object
- [fork / clone / exec / exit](fork-clone-exec-exit.md): 생성과 종료 경로
- [scheduler](scheduler.md): runnable, sleeping, zombie 같은 실행 상태
- [4. Object Lifetime](../4-object-lifetime/README.md): refcount, ownership, RCU 일반 원칙
- [5. Concurrency](../5-concurrency/README.md): exit race와 lock/RCU 보호

## 기억할 문장

task lifetime에서 가장 중요한 질문은 “task가 실행 중인가?”가 아니라 “이 task object를 지금 참조해도 안전한가?”다.
