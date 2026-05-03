# syzkaller

## 한 문장 정의

`syzkaller`는 syscall description과 coverage feedback을 이용해 kernel crash를 자동으로 찾고 reproducer를 생성하는 kernel fuzzer다.

## 왜 중요한가

커널 공격면은 syscall, ioctl, netlink, filesystem, socket option, bpf, io_uring처럼 넓다. 사람이 모든 조합을 직접 만들기는 어렵다. syzkaller는 syscall sequence를 생성하고, kernel coverage를 보며 더 많은 path를 탐색한다.

이 문서의 줄기는 이것이다.

```text
syzkaller는 user/kernel boundary를 자동으로 흔들어 kernel 상태 규칙이 깨지는 입력 조합을 찾는 도구다.
```

syzkaller의 가치는 crash 자체만이 아니라, crash를 반복할 수 있는 reproducer와 report를 남기는 데 있다.

## 기본 구성

syzkaller 환경은 보통 다음 구성으로 이루어진다.

- `syz-manager`: fuzzing 전체를 관리
- `syz-executor`: guest VM 안에서 syscall program 실행
- `syz-fuzzer`: corpus와 mutation을 관리
- `syz-repro`: crash reproducer 축소
- VM backend: QEMU 등
- kernel build: KCOV와 debug option이 포함된 image
- syscall description: syscall argument 구조 정의

각 구성은 서로 다른 역할을 가진다. kernel image만 준비한다고 fuzzing이 되는 것은 아니다.

## coverage-guided fuzzing

syzkaller는 KCOV를 통해 kernel coverage 정보를 얻는다. coverage가 늘어나는 syscall sequence는 corpus에 남고, 이후 mutation의 기반이 된다.

중요한 흐름은 다음이다.

```text
syscall program 생성
    -> VM에서 실행
    -> KCOV coverage 수집
    -> 새로운 path면 corpus에 저장
    -> crash 발생 시 report 저장
    -> reproducer 축소 시도
```

coverage는 “새로운 kernel path를 지났는가”를 알려준다. 하지만 coverage가 높다고 보안 영향이 큰 bug를 찾는다는 뜻은 아니다.

## syscall description

syzkaller는 syscall argument를 아무 byte나 넣는 방식으로만 동작하지 않는다. description을 기반으로 fd, pointer, struct, flag, resource 관계를 생성한다.

description이 중요한 이유는 다음이다.

- 올바른 fd type을 다음 syscall에 전달할 수 있다.
- netlink message 구조를 만들 수 있다.
- ioctl command별 argument 형태를 표현할 수 있다.
- resource 생성과 사용 관계를 유지할 수 있다.
- flag 조합과 enum 값을 제한할 수 있다.

description이 부정확하면 중요한 path에 도달하지 못하거나 의미 없는 input만 생성할 수 있다.

## crash report

syzkaller report에서 봐야 할 정보는 다음이다.

- crash title
- kernel commit과 config
- console output
- sanitizer 또는 warning report
- reproducer program
- C reproducer 여부
- VM log
- suspected duplicate 여부

title은 편의를 위한 이름이다. root cause는 report의 stack, reproducer, source code를 연결해 찾아야 한다.

## reproducer

syzkaller reproducer는 crash를 일으킨 syscall sequence를 줄인 결과다.

구분할 형태는 다음이다.

- syz program: syzkaller DSL 형태의 reproducer
- C reproducer: 사람이 컴파일해 실행하기 쉬운 형태
- raw log: 아직 축소되지 않은 crash 실행 기록

reproducer를 볼 때는 다음을 확인한다.

- crash가 같은 kernel config에서 반복되는가?
- 특정 CPU 수나 timing이 필요한가?
- namespace, capability, cgroup 설정이 필요한가?
- race를 유도하기 위해 여러 thread가 필요한가?
- reproducer가 crash 외에 환경 상태를 바꾸는가?

## triage

syzkaller가 찾은 crash를 분석할 때는 먼저 분류해야 한다.

분류 기준은 다음이다.

- memory bug: KASAN report 중심
- data race: KCSAN report 중심
- lock bug: lockdep report 중심
- warning: `WARN_ON`, `BUG_ON`, refcount warning 등
- hang: soft lockup, RCU stall, deadlock 가능성
- leak: memory leak 또는 reference leak
- info leak: uninitialized data 또는 kernel pointer 노출 가능성

분류 뒤 semantic tree의 어느 큰가지가 깨졌는지 연결한다.

## corpus와 sandbox

syzkaller는 VM 안에서 공격적인 syscall sequence를 실행한다. 격리와 상태 초기화가 중요하다.

확인할 항목은 다음이다.

- VM snapshot 또는 clean image 사용
- network 접근 제한
- rootfs 오염 여부
- crash 후 VM 재시작
- corpus 저장 위치
- target syscall enable/disable 설정
- sandbox mode와 namespace 설정

sandbox 설정은 도달 가능한 kernel path를 바꾼다. container 관련 bug를 찾을 때와 host-wide subsystem을 찾을 때 필요한 설정이 다를 수 있다.

## 코드에서 확인할 것

syzkaller report를 코드와 연결할 때는 다음을 찾는다.

- reproducer의 syscall sequence
- 각 syscall이 만드는 kernel object
- fd, socket, map, ring, namespace 같은 resource 연결
- crash stack의 object access
- reproducer에서 user가 조작하는 size, flag, index, pointer
- crash 전 cleanup 또는 close 경로
- concurrency를 유도하는 thread나 timing

## 보안 관점

syzkaller 관련 분석에서 주의할 점은 다음이다.

- crash title만 보고 중복이라고 단정하지 않기
- reproducer가 실제 권한 없이 실행 가능한지 확인
- sanitizer-only crash와 일반 kernel 영향 구분
- WARNING이 보안 영향이 없는지 별도로 확인
- race reproducer의 성공률과 timing 조건 기록
- fix 검증 시 같은 reproducer와 같은 config 사용
- report의 symptom과 root cause를 분리

검토할 질문은 다음과 같다.

1. reproducer는 어떤 user/kernel boundary를 통과하는가?
2. crash에 도달하기 위해 어떤 kernel object들이 만들어지는가?
3. 깨진 규칙은 lifetime, memory, permission, concurrency 중 무엇인가?
4. reproducer가 최소화되었어도 root cause에 필요한 syscall이 남아 있는가?
5. patch 후 같은 syz program과 C reproducer가 모두 통과하는가?

## 다른 문서와의 연결

- [build](build.md): syzkaller용 kernel은 KCOV, sanitizer, debug info config가 중요하다.
- [QEMU](qemu.md): syzkaller는 QEMU VM을 반복적으로 생성해 crash를 격리한다.
- [KASAN / KCSAN / UBSAN](kasan-kcsan-ubsan.md): syzkaller crash는 sanitizer report와 함께 분석하는 경우가 많다.
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): syzkaller는 syscall boundary를 자동으로 탐색한다.

## 기억할 문장

syzkaller는 crash를 대신 이해해주는 도구가 아니라, kernel boundary를 자동으로 탐색해 분석 가능한 reproducer와 증거를 만들어 주는 도구다.
