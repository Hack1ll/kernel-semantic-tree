# ftrace

## 한 문장 정의

`ftrace`는 커널 함수 호출, tracepoint, event, scheduling 흐름을 runtime에서 기록하는 tracing framework다.

## 왜 중요한가

코드만 읽으면 “이 path를 탈 것이다”라고 추정하게 된다. ftrace는 실제 workload가 어떤 kernel path를 지나갔는지 보여준다. crash가 없어도 상태 전이와 함수 순서를 관찰할 수 있다.

이 문서의 줄기는 이것이다.

```text
ftrace는 내가 예상한 kernel 실행 경로와 실제 실행 경로가 같은지 확인하는 도구다.
```

sanitizer가 규칙 위반을 report한다면, ftrace는 그 전후의 흐름을 좁히는 데 강하다.

## tracefs

ftrace는 보통 tracefs를 통해 제어한다.

자주 보는 파일은 다음이다.

- `current_tracer`
- `available_tracers`
- `set_ftrace_filter`
- `set_ftrace_notrace`
- `trace`
- `trace_pipe`
- `tracing_on`
- `events/`
- `buffer_size_kb`

이 파일들을 통해 어떤 함수를 추적할지, 어떤 event를 켤지, trace buffer를 언제 비울지 정한다.

## function tracer

function tracer는 함수 진입을 기록한다. 특정 subsystem의 path를 빠르게 확인할 때 유용하다.

확인할 수 있는 질문은 다음이다.

- syscall 뒤 실제로 어떤 kernel function이 호출되는가?
- driver의 file operation callback이 호출되는가?
- error path가 실행되는가?
- cleanup function이 호출되는가?
- worker나 timer callback으로 흐름이 넘어가는가?

필터를 좁히지 않으면 trace가 너무 커진다. 관심 함수 prefix나 정확한 function 이름으로 제한하는 것이 중요하다.

## function graph tracer

function graph tracer는 함수 진입과 반환을 함께 보여준다. 호출 관계와 duration을 보기 좋다.

유용한 상황은 다음이다.

- 특정 operation이 어느 하위 함수에서 오래 걸리는지 확인
- lock을 오래 잡는 구간 추정
- error return이 어디서 발생하는지 추적
- nested call 구조 파악

다만 overhead가 크고 output이 많다. 필요한 범위만 좁혀 쓰는 편이 좋다.

## tracepoint와 event

tracepoint는 kernel code에 미리 정의된 관찰 지점이다. ftrace는 event interface를 통해 tracepoint를 켜고 끌 수 있다.

대표 범주는 다음이다.

- sched events
- irq events
- workqueue events
- timer events
- filesystem events
- block events
- net events
- syscalls events

tracepoint는 함수 이름보다 안정적인 의미를 갖는 경우가 많다. 함수 이름이 바뀌어도 event의 의미가 유지될 수 있기 때문이다.

## kprobe와 dynamic event

ftrace 환경에서는 kprobe 기반 dynamic event를 만들어 특정 함수 인자나 return을 관찰할 수 있다.

확인할 지점은 다음이다.

- 함수가 실제로 호출되는가?
- 특정 인자 값이 무엇인가?
- return value가 무엇인가?
- 같은 object pointer가 여러 함수 사이에서 이어지는가?

단, optimized build나 inline, architecture별 calling convention 때문에 인자 관찰이 단순하지 않을 수 있다.

## trace 분석 순서

trace를 볼 때는 output을 많이 모으는 것보다 질문을 좁히는 것이 중요하다.

좋은 순서는 다음이다.

1. 확인할 path를 한 문장으로 정한다.
2. 시작 함수와 끝 함수를 고른다.
3. filter를 좁힌다.
4. trace buffer를 비운다.
5. workload 또는 reproducer를 실행한다.
6. timestamp, CPU, task 이름을 함께 본다.
7. 예상 경로와 다른 지점을 표시한다.

trace는 양이 많기 때문에 수집 전 질문이 중요하다.

## sanitizer report와 함께 쓰기

KASAN report가 access stack을 보여줘도 그 전에 어떤 syscall sequence가 object를 만들고 해제했는지는 부족할 수 있다. ftrace는 report 전후 경로를 보완한다.

예시는 다음과 같다.

- allocation function 호출 추적
- release function 호출 추적
- refcount get/put 호출 추적
- lock acquire/release 주변 함수 추적
- worker queue와 callback 실행 순서 추적

ftrace는 root cause를 증명하는 보조 증거로 자주 쓰인다.

## 코드에서 확인할 것

ftrace를 쓰기 전에 코드에서 다음을 정한다.

- trace할 entry function
- state를 바꾸는 function
- cleanup function
- callback function
- worker 또는 timer function
- user input이 들어오는 syscall/ioctl/netlink path
- object pointer가 전달되는 주요 함수

trace target을 코드에서 먼저 고르면 output을 훨씬 줄일 수 있다.

## 보안 관점

ftrace 자체는 버그 탐지기가 아니다. 하지만 취약점 분석에서 중요한 증거를 만든다.

- user input이 취약 path에 도달하는지 확인
- permission check가 실제로 실행되는지 확인
- cleanup path가 실패 경로에서 호출되는지 확인
- race 재현 중 두 CPU path의 순서 확인
- worker, timer, interrupt callback 실행 여부 확인
- patch 후 위험 path가 더 이상 실행되지 않는지 확인

검토할 질문은 다음과 같다.

1. 내가 예상한 function path가 실제 workload에서 실행되는가?
2. object를 생성한 path와 해제한 path가 trace에서 연결되는가?
3. permission check가 state change보다 먼저 호출되는가?
4. error path에서도 cleanup function이 호출되는가?
5. trace overhead가 timing-sensitive bug 재현을 바꾸지 않는가?

## 다른 문서와의 연결

- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): sanitizer report 전후의 실행 경로를 ftrace로 좁힐 수 있다.
- [lockdep](lockdep.md): lockdep report의 두 lock path를 trace로 확인할 수 있다.
- [QEMU](qemu.md): tracefs mount와 trace 저장은 VM 환경에서 준비해야 한다.
- [syzkaller](syzkaller.md): syzkaller reproducer가 어떤 kernel path를 지나는지 확인할 수 있다.

## 기억할 문장

ftrace는 crash 원인을 자동으로 말해주지 않지만, 실제 kernel이 어느 path를 지나갔는지 확인하게 해준다.
