# 11. eBPF

## 핵심 질문

user가 준 작은 프로그램을 커널 안에서 실행해도 안전하다고 어떻게 판단하는가?

## 큰가지의 의미

eBPF는 user space가 제공한 program을 커널 context에서 실행하게 한다. 따라서 핵심 경계는 verifier다. verifier가 “이 program은 안전하다”고 판단한 의미와 runtime/JIT가 실제 실행하는 의미가 같아야 한다.

```text
bpf syscall
    -> program / map 생성
    -> verifier
    -> helper / map access rule
    -> attach point
    -> runtime or JIT execution
```

## 하위 문서의 역할

- [verifier](verifier.md): BPF program의 pointer, range, control flow를 load 전에 검증한다.
- [maps](maps.md): BPF program과 user space가 공유하는 key-value kernel object
- [helpers](helpers.md): BPF program이 제한적으로 호출할 수 있는 kernel function
- [JIT](jit.md): BPF bytecode를 native code로 변환한다.
- [attach points](attach-points.md): BPF program이 실행되는 kernel hook 위치

## 이 장에서 특히 구분할 것

BPF는 일반 user input보다 강하다. user가 값만 주는 것이 아니라, 커널 안에서 실행될 instruction sequence를 준다.

그래서 세 모델이 일치해야 한다.

```text
verifier가 이해한 program
runtime interpreter가 실행하는 program
JIT가 생성한 native code
```

## 대표 흐름

```text
BPF program load
    -> instruction validation
    -> register state tracking
    -> helper permission check
    -> program fd 반환
```

```text
XDP attach
    -> packet receive
    -> BPF context 제공
    -> program 실행
    -> PASS / DROP / REDIRECT
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): `bpf()` syscall은 user-provided program을 받는 경계다.
- [3. Memory Management](../3-memory-management/README.md): verifier는 pointer와 range를 추적한다.
- [4. Object Lifetime](../4-object-lifetime/README.md): map value pointer와 bpf_link lifetime이 중요하다.
- [7. Permission Model](../7-permission-model/README.md): program type과 helper 사용에는 권한 제약이 붙는다.

## 보안 관점

eBPF 취약점은 verifier model과 runtime model의 차이에서 자주 나온다.

- range analysis 오류
- pointer type confusion
- helper return state 모델 오류
- map value lifetime 오류
- JIT sign extension 또는 bounds check 차이
- attach/detach race

## 읽고 나서 확인할 것

1. verifier는 이 pointer와 scalar range를 어떻게 이해하는가?
2. helper call 뒤 register state가 정확히 갱신되는가?
3. map value pointer lifetime은 어떻게 보장되는가?
4. JIT code가 verifier가 승인한 의미와 같은가?
5. attach point context에서 접근 가능한 data 범위는 어디까지인가?
