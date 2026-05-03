# scheduler

## 한 문장 정의

scheduler는 실행 가능한 task 중 어느 task를 어느 CPU에서 실행할지 결정하고, sleep, wakeup, context switch를 관리하는 커널 구성요소다.

## 왜 중요한가

CPU는 한 순간에 제한된 수의 task만 실행할 수 있다. 커널은 runnable 상태의 task를 CPU별 runqueue에 두고, 필요할 때 다음 task를 선택한다.

scheduler를 이해할 때 핵심은 “권한”이나 “fd”가 아니라 실행 상태다.

```text
runnable task
    -> runqueue에 있음
    -> CPU를 받으면 running
    -> I/O, lock, timer 조건으로 sleep
    -> 조건이 만족되면 wakeup
    -> 다시 runnable
```

이 흐름을 알아야 `current`가 바뀌는 시점, 잠들 수 있는 context, lock을 잡은 상태에서 호출하면 안 되는 함수들을 구분할 수 있다.

## 실행 상태

task는 계속 CPU를 쓰는 것이 아니다. 커널은 task를 여러 상태로 관리한다.

- running: 지금 CPU에서 실행 중이다.
- runnable: 실행할 준비가 되었지만 CPU를 기다린다.
- sleeping: I/O, lock, event, timer 같은 조건을 기다린다.
- stopped 또는 traced: signal, ptrace 같은 이유로 멈춰 있다.
- dead 또는 zombie: 실행은 끝났지만 정리 상태가 남아 있다.

상태 이름 자체보다 중요한 것은 전환 조건이다. 어떤 코드가 task를 재우고, 어떤 코드가 다시 깨우는지를 봐야 한다.

## runqueue와 CPU 선택

각 CPU는 실행 가능한 task 목록을 관리한다. scheduler는 priority, scheduling class, CPU affinity, load balancing 같은 조건을 보고 다음 task를 고른다.

```text
task wakeup
    -> runnable 상태로 변경
    -> 적절한 CPU runqueue에 배치
    -> scheduler가 pick next
    -> context switch
```

context switch가 일어나면 CPU에서 실행 중인 `current`가 바뀐다. 다만 각 task의 `task_struct`와 연결된 `mm`, `files`, `cred`가 사라지는 것은 아니다. 실행 주체가 바뀌는 것이다.

## sleep과 wakeup

kernel code는 조건이 충족되지 않으면 task를 sleep 상태로 둘 수 있다. 대표 상황은 다음과 같다.

- blocking I/O가 완료될 때까지 기다린다.
- mutex를 얻을 때까지 기다린다.
- wait queue에서 event를 기다린다.
- timer 만료를 기다린다.

sleep 가능한 context와 불가능한 context를 구분해야 한다. spinlock을 잡은 상태, interrupt context, preemption이 꺼진 구간에서는 일반적인 sleep을 하면 안 된다.

## preemption

preemption은 실행 중인 task가 스스로 CPU를 내려놓지 않아도 kernel이 다른 task로 전환할 수 있게 하는 규칙이다. preemption 가능 여부는 lock, interrupt, kernel config, 실행 context에 영향을 받는다.

커널 코드를 읽을 때는 다음을 구분한다.

- 이 구간에서 schedule이 호출될 수 있는가?
- 이 구간에서 preemption이 막혀 있는가?
- lock을 잡은 동안 sleep 가능한 함수를 호출하는가?
- CPU-local data를 다룰 때 다른 CPU와의 경쟁이 있는가?

## scheduler가 직접 다루는 것과 아닌 것

scheduler는 CPU 시간을 배분한다. 하지만 task의 권한을 결정하거나 fd table을 직접 해석하지 않는다.

- 권한 판단: [7. Permission Model](../7-permission-model/README.md)
- fd 해석: [6. VFS / FD Model](../6-vfs-fd-model/README.md)
- task 구조: [task_struct](task-struct.md)
- task 종료 후 해제: [task lifetime](task-lifetime.md)

scheduler 문서를 다른 문서와 분리해서 읽어야 하는 이유가 여기에 있다.

## 코드에서 확인할 것

1. 이 함수는 sleep 가능한 context에서 호출되는가?
2. task state를 바꾼 뒤 wakeup 경로가 반드시 존재하는가?
3. wait queue에 등록한 뒤 조건 검사 순서가 올바른가?
4. lock을 잡은 상태에서 schedule 가능성이 있는 함수를 호출하는가?
5. per-CPU runqueue나 CPU-local data를 접근할 때 보호 규칙이 있는가?
6. context switch 뒤에도 유지된다고 가정한 값이 task-local 값인가, CPU-local 값인가?

## 보안/안정성 관점

scheduler 자체보다 scheduler와 다른 큰가지가 만나는 지점에서 문제가 생기기 쉽다.

- sleep 불가능한 context에서 sleep을 시도해 커널 오류가 난다.
- wakeup 누락으로 task가 영원히 깨어나지 못한다.
- wait condition 검사와 sleep 등록 순서가 틀려 event를 놓친다.
- preemption이 가능한 구간에서 보호되지 않은 shared state를 만진다.
- context switch 전후로 `current`나 CPU-local pointer 의미를 혼동한다.

## 기억할 문장

scheduler는 task의 권한이나 자원을 설명하는 장치가 아니라, task가 언제 CPU를 얻고 언제 멈추는지를 설명하는 장치다.
