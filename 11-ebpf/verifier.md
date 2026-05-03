# verifier

## 한 문장 정의

`verifier`는 BPF 프로그램을 실행하기 전에 모든 실행 경로를 분석해, 커널 메모리와 커널 객체를 안전한 방식으로만 다루는지 판단하는 정적 분석기다.

## 왜 중요한가

eBPF의 핵심 위험은 user space가 만든 instruction sequence가 커널 권한으로 실행된다는 점이다. 커널은 이 코드를 그대로 믿을 수 없다. 그래서 `BPF_PROG_LOAD` 단계에서 verifier가 프로그램을 거부하거나 승인한다.

verifier가 승인한다는 말은 단순히 문법이 맞는다는 뜻이 아니다.

```text
이 프로그램의 모든 실행 경로에서
잘못된 pointer 접근이 없고,
허용되지 않은 helper 호출이 없고,
stack과 packet/map/context 접근 범위가 안전하고,
종료 조건과 상태 변화가 분석 가능하다는 뜻이다.
```

따라서 verifier를 볼 때의 줄기는 이것이다.

```text
verifier는 BPF 프로그램의 실제 실행을 abstract state로 먼저 실행해 보는 장치다.
```

## load 경로에서의 위치

대표 흐름은 다음과 같다.

```text
bpf(BPF_PROG_LOAD)
    -> instruction 복사
    -> program type 확인
    -> verifier 실행
    -> helper, map, context 접근 규칙 확인
    -> JIT 또는 interpreter 실행 준비
```

verifier는 program이 attach되기 전에 동작한다. attach point에서 packet이나 trace event가 발생한 뒤에 검사하는 구조가 아니다.

## verifier가 추적하는 것

verifier가 보는 핵심 단위는 register state다. BPF register는 단순한 숫자일 수도 있고, 특정 kernel object를 가리키는 pointer일 수도 있다.

대표적으로 구분해야 하는 상태는 다음과 같다.

- `SCALAR_VALUE`: 일반 정수 값
- `PTR_TO_CTX`: attach point가 제공한 context pointer
- `PTR_TO_PACKET`: packet data pointer
- `PTR_TO_PACKET_END`: packet 끝을 나타내는 pointer
- `PTR_TO_MAP_VALUE`: map element value pointer
- `PTR_TO_STACK`: BPF stack pointer
- `PTR_TO_BTF_ID`: BTF type이 붙은 kernel object pointer

여기서 중요한 점은 “값”과 “pointer”가 같은 register에 들어갈 수 있지만, verifier는 둘을 다르게 취급한다는 것이다. pointer에 임의 산술 연산을 허용하면 커널 메모리 전체가 접근 대상이 될 수 있기 때문이다.

## range analysis

verifier는 정수 값의 범위를 추적한다.

```text
if (idx < 16)
    value = array[idx];
```

이런 코드는 `idx`가 실제로 `0..15` 범위에 있다는 사실을 verifier가 이해해야 안전하게 통과된다.

분석 대상은 signed/unsigned 범위, 32-bit/64-bit 연산, bitmask, shift, add/sub 결과까지 이어진다. 이 부분이 틀리면 verifier는 위험한 offset을 안전하다고 판단할 수 있다.

취약점 연구에서 자주 보는 질문은 다음이다.

- signed 비교 뒤 unsigned 접근을 해도 범위가 유지되는가?
- 32-bit ALU 연산 뒤 상위 32-bit 상태가 정확한가?
- overflow 가능성이 range에 반영되는가?
- branch merge 후 더 넓은 범위가 필요한데 좁게 남지 않는가?

## pointer bounds check

packet이나 map value를 읽으려면 먼저 경계를 증명해야 한다.

```text
data + offset + size <= data_end
```

verifier는 이런 형태의 check를 보고 이후 접근을 허용한다. 단, check한 pointer와 실제 접근 pointer가 의미상 같아야 한다.

보안 관점에서 위험한 지점은 다음이다.

- check한 offset과 사용하는 offset이 달라지는 경우
- helper 호출 뒤 packet pointer가 무효화되었는데 계속 쓰는 경우
- map value size보다 큰 offset을 verifier가 놓치는 경우
- pointer에 scalar가 더해진 뒤 범위 정보가 과하게 유지되는 경우

## stack 검증

BPF program은 제한된 stack을 가진다. verifier는 stack slot 단위로 초기화 여부와 pointer 저장 여부를 추적한다.

확인해야 할 규칙은 다음이다.

- 초기화되지 않은 stack byte를 읽으면 안 된다.
- kernel pointer가 user로 leak될 수 있는 경로가 없어야 한다.
- helper에 넘기는 stack buffer의 크기와 초기화 범위가 맞아야 한다.
- stack에 저장한 pointer를 다시 읽을 때 type 정보가 유지되어야 한다.

stack 검증이 약하면 uninitialized memory leak이나 type confusion으로 이어질 수 있다.

## helper 호출 검증

helper는 BPF program이 호출할 수 있는 커널 함수다. verifier는 helper prototype을 알고 있어야 한다.

검사하는 내용은 다음과 같다.

- 이 program type에서 호출 가능한 helper인가?
- 각 인자의 type이 helper prototype과 맞는가?
- pointer 인자가 가리키는 메모리 범위가 충분한가?
- helper 호출 후 register state가 어떻게 바뀌는가?
- helper가 기존 pointer를 무효화하는가?

helper의 실제 side effect와 verifier가 모델링한 side effect가 다르면 취약점이 생긴다.

## control flow와 state pruning

verifier는 모든 branch를 따라가야 한다. 하지만 모든 path를 무한정 저장할 수는 없기 때문에, 같은 지점에 도달한 상태를 비교하고 일부 상태를 pruning한다.

여기서 핵심은 상태를 합치거나 버릴 때 의미가 보존되어야 한다는 점이다.

위험한 질문은 다음이다.

- 두 state가 실제로 다른데 같은 state로 판단하지 않는가?
- loop bound가 충분히 엄격하게 제한되는가?
- unreachable path로 보이는 곳에 실제 실행 가능성이 남아 있지 않은가?
- pruning 후 pointer range가 과하게 좁아지지 않는가?

## 코드에서 확인할 것

verifier 관련 코드를 읽을 때는 다음 객체를 찾는다.

- `struct bpf_verifier_env`: verifier 전체 실행 상태
- `struct bpf_verifier_state`: 특정 instruction 지점의 program state
- `struct bpf_func_state`: 함수 호출 단위의 register/stack 상태
- `struct bpf_reg_state`: register 하나의 type, range, id, offset
- helper prototype table: program type별 helper 허용 규칙

중요한 흐름은 “instruction 하나를 해석한 뒤 abstract state가 어떻게 바뀌는가”다.

## 보안 관점

verifier 취약점은 보통 다음 형태로 나타난다.

- unsafe pointer arithmetic을 safe로 판단
- signed/unsigned range 처리 오류
- 32-bit ALU 결과의 상위 bit 모델링 오류
- helper side effect 모델링 누락
- state pruning이 서로 다른 상태를 같은 상태로 취급
- stack 초기화 추적 실패
- map value pointer의 lifetime 또는 size 추적 오류

검증 질문은 다음과 같다.

1. verifier가 승인한 pointer 접근은 runtime에서도 같은 범위 안에 있는가?
2. 모든 branch와 loop에서 같은 safety invariant가 유지되는가?
3. helper 호출 전후 register state가 실제 helper 동작과 일치하는가?
4. JIT가 verifier가 승인한 instruction 의미를 그대로 실행하는가?
5. map, context, packet pointer가 서로 다른 type으로 섞이지 않는가?

## 다른 문서와의 연결

- [maps](maps.md): `PTR_TO_MAP_VALUE`가 가리키는 실제 storage와 lifetime을 다룬다.
- [helpers](helpers.md): verifier가 helper prototype과 side effect를 어떻게 믿는지 이어서 본다.
- [JIT](jit.md): verifier가 승인한 의미가 native code에서 보존되는지 확인한다.
- [attach points](attach-points.md): `PTR_TO_CTX`의 구조와 허용 접근은 attach point가 정한다.

## 기억할 문장

verifier는 BPF program을 실행하지 않고도, 실행 가능한 모든 경로가 커널 안전 규칙을 깨지 않는다고 증명해야 한다.
