# 13. Debugging / Testing

## 핵심 질문

내가 이해한 kernel object, lifetime, permission, concurrency 규칙이 실제 실행에서도 맞는지 어떻게 확인하는가?

## 큰가지의 의미

커널은 읽기만으로 끝나는 대상이 아니다. build하고, VM에서 부팅하고, sanitizer와 tracing을 켜고, reproducer를 실행해야 실제 상태 전이를 볼 수 있다.

이 장은 지식 확인 도구를 다룬다.

```text
hypothesis
    -> kernel config
    -> build
    -> QEMU boot
    -> instrumentation
    -> reproducer
    -> report / trace
    -> root cause
```

## 하위 문서의 역할

- [build](build.md): 원하는 config로 kernel image를 만드는 과정
- [QEMU](qemu.md): 실험용 kernel을 격리된 VM에서 부팅하는 도구
- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): memory bug, race, undefined behavior를 runtime에 탐지하는 sanitizer
- [lockdep](lockdep.md): lock ordering과 context rule 위반을 찾는 도구
- [ftrace](ftrace.md): kernel function과 event 흐름을 tracing하는 framework
- [syzkaller](syzkaller.md): syscall sequence를 자동 생성해 crash를 찾는 fuzzer

## 이 장에서 특히 구분할 것

도구마다 답하는 질문이 다르다.

```text
build   -> 이 config와 source가 실행 가능한가?
QEMU    -> crash를 안전하게 반복할 수 있는가?
KASAN   -> memory access가 잘못되었는가?
KCSAN   -> data race 가능성이 있는가?
lockdep -> lock order와 context가 맞는가?
ftrace  -> 실제로 어떤 함수 경로를 지나갔는가?
syzkaller -> 어떤 syscall 조합이 bug를 드러내는가?
```

## 대표 흐름

```text
KASAN UAF 분석
    -> KASAN config로 build
    -> QEMU boot
    -> reproducer 실행
    -> allocation/free/use stack 확인
    -> root cause path 추적
```

```text
ftrace 경로 확인
    -> 관심 function filter 설정
    -> workload 실행
    -> trace buffer에서 호출 순서 확인
```

## 다른 큰가지와의 연결

- [3. Memory Management](../3-memory-management/README.md): KASAN은 memory bug를 관찰한다.
- [4. Object Lifetime](../4-object-lifetime/README.md): UAF report는 lifetime 위반의 증거가 된다.
- [5. Concurrency](../5-concurrency/README.md): KCSAN과 lockdep은 race와 lock 문제를 좁힌다.
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syzkaller는 syscall boundary를 체계적으로 흔든다.

## 보안 관점

debugging output은 결론이 아니라 증거다.

- sanitizer report는 증상 위치를 보여준다.
- ftrace는 실제 실행 경로를 보여준다.
- lockdep은 deadlock 가능성을 보여준다.
- syzkaller reproducer는 trigger를 보여준다.

root cause는 object lifetime, permission, memory, concurrency 규칙 중 무엇이 깨졌는지 다시 연결해야 보인다.

## 읽고 나서 확인할 것

1. 내가 확인하려는 가설은 무엇인가?
2. 어떤 config와 debug option이 필요한가?
3. crash log와 trace를 저장할 수 있는가?
4. reproducer가 같은 환경에서 반복되는가?
5. report의 증상을 semantic tree의 어떤 큰가지로 연결할 수 있는가?
