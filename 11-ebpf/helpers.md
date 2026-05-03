# helpers

## 한 문장 정의

`helper`는 BPF program이 직접 할 수 없는 커널 작업을 제한된 형태로 요청하기 위해 호출하는 커널 제공 함수다.

## 왜 중요한가

BPF program은 임의의 커널 함수를 호출할 수 없다. 호출 가능한 함수 목록은 program type과 attach point에 따라 제한된다. 이 제한된 호출 지점이 helper다.

helper를 이해할 때의 줄기는 이것이다.

```text
helper는 BPF program과 커널 내부 기능 사이의 허가된 함수 호출 경계다.
```

따라서 helper의 핵심은 함수 이름이 아니라 계약이다.

```text
어떤 program type에서 호출 가능한가?
각 인자의 type은 무엇이어야 하는가?
호출 후 pointer와 register state는 어떻게 바뀌는가?
실제 helper 구현은 verifier가 믿은 계약을 지키는가?
```

## 호출 가능성 결정

BPF instruction에는 helper id가 들어간다. 커널은 program type별로 이 id가 어떤 helper를 가리키는지 확인한다.

대표 흐름은 다음과 같다.

```text
BPF_CALL instruction
    -> helper id 확인
    -> program type별 helper prototype 조회
    -> argument type 검증
    -> return type을 verifier state에 반영
    -> runtime에서 실제 helper 함수 호출
```

같은 helper라도 모든 program type에서 허용되지 않는다. packet 경로에서만 의미 있는 helper, tracing에서만 의미 있는 helper, sleep 가능한 context에서만 호출 가능한 helper가 따로 있다.

## helper prototype

verifier는 helper의 prototype을 보고 인자를 검사한다.

prototype에는 대략 다음 정보가 들어간다.

- helper return type
- 각 argument의 expected type
- memory pointer 인자의 size argument
- map pointer 인자의 허용 map type
- context pointer 사용 가능 여부
- helper 호출 후 변경되는 state

예를 들어 helper가 memory buffer와 size를 받는다면, verifier는 pointer가 유효한지와 size만큼 접근 가능한지 확인해야 한다.

## 대표 helper 범주

helper는 기능별로 나누어 보는 편이 좋다.

- map helper: `bpf_map_lookup_elem`, `bpf_map_update_elem`, `bpf_map_delete_elem`
- packet helper: packet head/tail 조정, checksum 보정, redirect
- tracing helper: kernel memory read, current task 정보 조회, event 출력
- time/helper: time, random, CPU id 조회
- ring buffer helper: reserve, submit, discard
- tail call helper: `BPF_MAP_TYPE_PROG_ARRAY`를 이용한 program 전환
- socket helper: socket lookup, redirect, local storage 접근

각 범주는 접근하는 kernel object가 다르다. map helper는 map lifetime, packet helper는 packet pointer invalidation, tracing helper는 kernel pointer read 규칙과 연결된다.

## 인자 검증

helper 인자 검증에서 중요한 것은 pointer의 종류다.

예시는 다음과 같다.

- map pointer: 실제 `struct bpf_map` reference가 있어야 한다.
- map value pointer: value 범위 안의 pointer여야 한다.
- stack pointer: 초기화된 stack 영역이어야 한다.
- packet pointer: `data_end` bounds check를 통과해야 한다.
- context pointer: attach point에서 허용된 field만 접근해야 한다.
- constant size: verifier가 상수로 알 수 있어야 하는 경우가 있다.

helper가 요구하는 type과 register의 verifier state가 맞지 않으면 program load가 실패해야 한다.

## return state

helper 호출 결과는 verifier state에 반영된다.

대표적인 반환 의미는 다음이다.

- 일반 scalar 값
- NULL일 수 있는 pointer
- map value pointer
- socket 또는 task 같은 reference-counted object pointer
- error code

반환 pointer가 NULL 가능성을 가지면 NULL check 전 접근이 금지된다. reference를 얻는 helper라면 release helper 또는 cleanup 경로가 맞아야 한다.

## side effect

helper는 단순히 값을 반환하는 함수가 아니다. 일부 helper는 기존 pointer 상태를 무효화하거나 kernel object 상태를 바꾼다.

중요한 side effect는 다음이다.

- packet data 위치 변경
- packet length 변경
- map element update 또는 delete
- ring buffer record 제출
- tail call로 현재 program 실행 흐름 전환
- reference 획득 또는 해제
- sleep 가능 helper에서 scheduling 발생

verifier가 이 side effect를 정확히 모델링하지 못하면 승인된 program의 실제 의미가 달라질 수 있다.

## kfunc와의 차이

최근 BPF는 helper 외에도 BTF type 정보를 이용한 `kfunc` 호출을 제공한다. helper는 고정된 helper id와 prototype table을 중심으로 동작하고, kfunc는 BTF 기반 type 정보와 allowlist를 통해 호출 가능성을 정한다.

이 문서는 helper를 중심으로 설명한다. kfunc를 볼 때도 같은 질문을 적용하면 된다.

```text
호출 가능한가?
인자 type이 맞는가?
반환 pointer와 reference는 어떻게 관리되는가?
side effect가 verifier에 반영되는가?
```

## 코드에서 확인할 것

helper 관련 코드를 읽을 때는 다음을 확인한다.

- program type별 helper 허용 table
- helper prototype을 반환하는 함수
- argument type 검사 로직
- helper implementation이 실제로 접근하는 kernel object
- helper 호출 후 register state 갱신 로직
- helper가 reference를 얻는지 또는 기존 pointer를 무효화하는지

## 보안 관점

helper 관련 취약점은 주로 verifier와 helper 구현의 계약 불일치에서 나온다.

- helper prototype이 실제 구현보다 느슨한 경우
- size argument 검증이 부족한 경우
- helper가 packet layout을 바꿨는데 verifier가 pointer를 그대로 유지하는 경우
- reference 획득 helper와 release helper의 짝이 깨지는 경우
- helper가 허용되지 않은 program type에서 호출 가능한 경우
- kernel memory read helper가 출력 범위를 잘못 제한하는 경우
- sleep 불가능 context에서 sleep 가능 helper가 호출되는 경우

검증 질문은 다음과 같다.

1. helper 인자의 verifier type과 실제 C 함수가 기대하는 type이 같은가?
2. helper가 접근하는 memory 범위가 verifier가 확인한 범위 안에 있는가?
3. helper 호출 뒤 무효화해야 하는 pointer가 남아 있지 않은가?
4. helper가 반환한 reference는 모든 경로에서 release되는가?
5. program type과 attach point가 이 helper의 실행 context를 감당할 수 있는가?

## 다른 문서와의 연결

- [verifier](verifier.md): helper 인자와 반환값을 abstract state로 검증하는 과정을 다룬다.
- [maps](maps.md): map helper가 조작하는 실제 객체와 lifetime을 확인한다.
- [attach points](attach-points.md): helper 허용 여부는 실행 위치와 context에 따라 달라진다.
- [JIT](jit.md): helper call은 native calling convention으로 변환되어 실행된다.

## 기억할 문장

helper는 BPF program에 허용된 커널 함수 호출이며, 안전성은 prototype, side effect, 실행 context가 모두 일치할 때 유지된다.
