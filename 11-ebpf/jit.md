# JIT

## 한 문장 정의

`JIT`는 verifier를 통과한 BPF bytecode를 CPU architecture별 native instruction으로 변환하는 실행 단계다.

## 왜 중요한가

BPF program은 interpreter로도 실행될 수 있지만, 성능이 중요한 경로에서는 native code로 변환된다. XDP, tracing, socket filtering 같은 경로에서 BPF는 매우 자주 실행될 수 있다.

JIT를 이해할 때의 줄기는 이것이다.

```text
JIT는 verifier가 안전하다고 승인한 BPF 의미를 CPU instruction으로 보존해야 한다.
```

즉 JIT의 핵심은 빠른 코드 생성만이 아니다. verifier가 분석한 의미와 실제 CPU가 실행하는 의미가 같아야 한다.

## 실행 준비 흐름

대표 흐름은 다음과 같다.

```text
BPF program load
    -> verifier 통과
    -> JIT 가능 여부 확인
    -> architecture별 code generation
    -> helper call과 branch target fixup
    -> executable memory에 배치
    -> attach point에서 native code 실행
```

JIT는 verifier를 대체하지 않는다. verifier가 승인한 program을 실행 가능한 기계어로 바꾸는 단계다.

## 보존해야 하는 의미

BPF instruction은 architecture와 무관한 의미를 가진다. JIT는 이를 각 CPU의 instruction set으로 옮긴다.

중요한 보존 대상은 다음이다.

- 64-bit ALU 연산 의미
- 32-bit ALU 연산 후 상위 bit 처리
- signed/unsigned 비교
- endianness 변환
- load/store width
- stack offset 계산
- branch offset 계산
- helper call ABI
- tail call 동작
- atomic operation 의미

이 중 하나라도 interpreter 의미와 달라지면 verifier가 승인한 안전성이 깨질 수 있다.

## register와 stack mapping

BPF에는 고정된 virtual register가 있다. JIT는 이를 실제 CPU register나 stack slot에 배치한다.

확인할 질문은 다음이다.

- BPF register 값이 helper call 전후에 보존되는가?
- caller-saved/callee-saved register 규칙이 architecture ABI와 맞는가?
- BPF stack 접근 offset이 native stack 위치로 정확히 변환되는가?
- tail call이나 exception path에서 stack frame이 깨지지 않는가?
- 32-bit register write 후 zero-extension 처리가 맞는가?

JIT 버그는 C 코드의 일반 버그라기보다, 두 instruction set 사이의 의미 차이에서 많이 발생한다.

## branch와 offset

BPF instruction은 branch offset을 가진다. JIT는 native instruction 크기를 계산한 뒤 실제 jump target을 맞춰야 한다.

위험한 지점은 다음이다.

- long jump와 short jump 변환
- branch target 계산 overflow
- dead code 제거 후 offset 재계산
- exception table 또는 fixup table과의 연결
- tail call limit 처리

branch target이 틀리면 verifier가 분석하지 않은 instruction이 실행될 수 있다.

## helper call 변환

BPF helper call은 native function call로 변환된다. 이때 architecture calling convention을 지켜야 한다.

확인할 항목은 다음이다.

- helper 인자가 올바른 native register로 전달되는가?
- return value가 BPF register `R0` 의미로 돌아오는가?
- helper가 clobber할 수 있는 register가 보존 또는 폐기되는가?
- speculation mitigation이나 indirect call 제한이 필요한가?
- helper trampoline이나 kfunc call 경로와 ABI가 맞는가?

helper 자체의 의미는 [helpers](helpers.md)에서 다루고, 이 문서에서는 그 호출이 native code로 올바르게 표현되는지를 본다.

## JIT hardening

JIT code는 실행 가능한 커널 메모리에 놓인다. 따라서 성능 외에도 공격면이 된다.

주요 hardening 포인트는 다음이다.

- code memory를 write 가능한 상태로 오래 두지 않기
- 생성 후 read-only/executable 속성 적용
- constant blinding으로 JIT spraying 위험 줄이기
- branch target과 call target 제한
- speculation 관련 barrier 또는 masking
- unprivileged BPF 설정과 JIT hardening 설정의 상호작용

hardening은 architecture별 구현 차이가 크다. 같은 BPF program이라도 CPU별 JIT code는 다르게 생성된다.

## interpreter와의 관계

JIT가 꺼져 있거나 특정 조건에서 사용할 수 없으면 interpreter가 실행된다. 보안 관점에서는 두 실행 방식의 의미가 같아야 한다.

검증 질문은 다음이다.

- interpreter는 정상인데 JIT에서만 다른 결과가 나오지 않는가?
- JIT에서만 sign extension 또는 zero extension이 달라지지 않는가?
- JIT에서 helper call 인자 배치가 interpreter와 다른 의미를 만들지 않는가?
- JIT hardening이 원래 BPF 의미를 바꾸지 않는가?

## 코드에서 확인할 것

JIT 코드를 읽을 때는 architecture별 backend를 본다.

확인할 항목은 다음이다.

- BPF instruction별 emit 함수
- register mapping table
- prologue와 epilogue 생성
- helper call emit 경로
- branch offset fixup 경로
- constant blinding 또는 hardening pass
- executable memory allocation과 permission 변경

## 보안 관점

JIT 관련 취약점은 주로 verifier와 runtime 의미의 차이에서 나온다.

- 32-bit ALU zero-extension 누락
- signed/unsigned branch 변환 오류
- load/store size 변환 오류
- branch target 계산 오류
- helper call ABI mismatch
- constant blinding이 instruction 의미를 바꾸는 경우
- JIT code memory permission 처리 오류
- speculation mitigation 누락

검증 질문은 다음과 같다.

1. 이 BPF instruction의 interpreter 의미와 JIT output 의미가 같은가?
2. verifier가 의존한 bounds check가 native code에도 같은 순서로 남아 있는가?
3. helper call 전후 register clobber 처리가 정확한가?
4. JIT hardening pass가 branch, constant, offset을 잘못 바꾸지 않는가?
5. 생성된 code memory는 필요한 시점 이후 쓰기 가능 상태로 남지 않는가?

## 다른 문서와의 연결

- [verifier](verifier.md): JIT는 verifier가 승인한 의미를 보존해야 한다.
- [helpers](helpers.md): helper 호출은 native ABI로 변환된다.
- [attach points](attach-points.md): JIT code는 attach point event에서 반복 실행된다.
- [3. Memory Management](../3-memory-management/README.md): executable kernel memory와 permission 변경을 연결해서 본다.

## 기억할 문장

JIT의 보안 기준은 빠르게 실행되는지가 아니라, verifier가 증명한 BPF 의미를 native code가 정확히 보존하는지다.
