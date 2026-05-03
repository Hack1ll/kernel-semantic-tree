# attach points

## 한 문장 정의

`attach point`는 BPF program이 연결되어 실제로 실행되는 커널 hook 또는 event 위치다.

## 왜 중요한가

같은 BPF instruction이라도 어디에 붙어서 실행되는지에 따라 의미가 달라진다. packet 경로, tracing 경로, cgroup 경로, LSM 경로는 제공하는 context, 허용 helper, 반환값 의미, 실행 가능한 상황이 다르다.

attach point를 이해할 때의 줄기는 이것이다.

```text
attach point는 BPF program의 실행 context와 권한 범위를 정한다.
```

verifier가 `ctx` pointer를 검증할 때도 attach point가 정한 context 구조를 기준으로 삼는다.

## program type과 attach type

BPF program은 load될 때 program type을 가진다. 일부 program은 추가로 expected attach type을 가진다.

구분할 항목은 다음이다.

- `program type`: program의 큰 실행 범주
- `attach type`: 같은 program type 안에서 붙는 구체 hook 종류
- `ctx`: program에 전달되는 context 구조
- return value 의미: 통과, 차단, redirect, error code 등
- helper allowlist: 이 위치에서 호출 가능한 helper 집합

program type과 attach type을 섞어 이해하면 `ctx` 접근과 helper 허용 규칙을 잘못 읽게 된다.

## 대표 attach point 범주

eBPF가 붙는 위치는 매우 다양하다. 큰 범주는 다음과 같다.

- XDP: network driver receive 경로의 빠른 packet 처리
- TC: traffic control ingress/egress 경로
- socket filter: socket으로 들어오는 packet 필터링
- cgroup hooks: cgroup 단위 network, process, device 관련 정책
- kprobe/kretprobe: kernel function 진입과 반환 지점 관찰
- uprobe/uretprobe: user process function 진입과 반환 지점 관찰
- tracepoint/raw tracepoint: kernel trace event에 연결
- fentry/fexit: BTF 정보를 이용한 function entry/exit 연결
- LSM: security hook에서 정책 실행
- perf event: performance event와 연결
- struct_ops: kernel operation table 일부를 BPF program으로 제공

각 범주는 서로 다른 context와 return rule을 가진다.

## attach 흐름

대표 흐름은 다음과 같다.

```text
BPF_PROG_LOAD
    -> program type과 attach type 검증
    -> verifier가 context 접근 규칙 적용
    -> attach 요청
    -> kernel hook에 program reference 연결
    -> event 발생 시 program 실행
    -> detach 또는 link close 시 연결 해제
```

attach 방식은 영역마다 다르다. `bpf_link`를 쓰는 경로도 있고, network subsystem이나 perf event API를 통해 붙는 경로도 있다.

## context

BPF program의 첫 번째 인자는 보통 context pointer다. 이 context가 무엇인지는 attach point가 정한다.

예시는 다음과 같다.

- XDP: packet data, data_end, ingress device 정보
- TC: `sk_buff` 기반 packet context
- tracepoint: trace event field layout
- kprobe: register state 또는 function argument
- LSM: security hook argument
- cgroup socket hook: socket 또는 socket address 관련 context

context 접근은 verifier가 제한한다. field를 읽을 수 있는지, 쓸 수 있는지, offset이 유효한지는 attach point별 규칙을 따른다.

## 반환값 의미

attach point마다 return value 의미가 다르다.

예시는 다음과 같다.

- XDP: `XDP_PASS`, `XDP_DROP`, `XDP_TX`, `XDP_REDIRECT`
- TC: packet 통과, drop, redirect 등 action
- socket filter: packet에서 유지할 byte 수
- cgroup hook: 허용 또는 거부
- LSM hook: 보안 결정을 나타내는 error code
- tracing hook: 반환값이 관찰 목적이거나 제한된 의미만 가짐

return value를 잘못 이해하면 program이 관찰용인지 정책 결정용인지 구분하지 못한다.

## sleep 가능 여부

모든 attach point에서 sleep이 가능한 것은 아니다. packet fast path나 interrupt에 가까운 경로에서는 blocking 동작이 허용되지 않는다. 반대로 sleepable BPF가 허용되는 경로에서는 일부 helper나 kfunc 호출 가능성이 넓어진다.

확인할 질문은 다음이다.

- 이 attach point는 sleep 가능한 context인가?
- helper가 blocking 동작을 할 수 있는가?
- RCU read-side context에서 실행되는가?
- preemption이나 migration에 대한 가정이 있는가?
- 여러 CPU에서 같은 program이 동시에 실행될 수 있는가?

## attach lifetime

program이 load되었다고 자동으로 실행되는 것은 아니다. attach되어야 event에서 호출된다.

lifetime을 결정하는 요소는 다음이다.

- program fd
- attach target object
- `bpf_link` fd
- bpffs에 pin된 link 또는 program
- network device, cgroup, perf event, tracing target의 lifetime
- detach path와 cleanup path

위험한 지점은 attach target이 사라지는 경로와 program detach 경로가 겹칠 때다.

## namespace와 권한

attach point는 권한 모델과도 연결된다. 예를 들어 network 관련 attach는 network namespace, cgroup 관련 attach는 cgroup hierarchy, tracing 관련 attach는 커널 관찰 권한과 연결된다.

확인할 질문은 다음이다.

- 어떤 namespace 기준으로 attach 권한을 검사하는가?
- target object가 user가 접근 가능한 범위 안에 있는가?
- attach 후 target이 다른 namespace로 이동하거나 사라질 수 있는가?
- unprivileged BPF 설정에서 이 attach가 허용되는가?
- LSM이나 capability check가 모든 attach 경로에 적용되는가?

## 코드에서 확인할 것

attach point 관련 코드를 읽을 때는 다음을 확인한다.

- program type과 attach type 검증 위치
- context access validation 함수
- helper allowlist 선택 함수
- attach target object reference 획득 위치
- detach와 cleanup 경로
- return value를 해석하는 caller
- event 발생 시 program을 호출하는 hot path

## 보안 관점

attach point 관련 취약점은 다음 형태로 나타난다.

- 잘못된 context field 접근 허용
- attach 권한 검사 누락
- target object lifetime 처리 오류
- detach와 event 실행의 race
- return value 의미 혼동으로 인한 정책 우회
- sleep 불가능 context에서 blocking helper 허용
- namespace 기준이 잘못된 attach 허용
- tracing hook에서 kernel pointer 또는 민감 정보 노출

검증 질문은 다음과 같다.

1. 이 program은 어떤 kernel event에서 실행되는가?
2. `ctx` pointer가 실제로 어떤 구조를 가리키는가?
3. 이 attach point에서 허용되는 helper와 금지되는 helper는 무엇인가?
4. attach target object는 program 실행 중 살아 있는가?
5. return value가 kernel caller에서 어떻게 해석되는가?

## 다른 문서와의 연결

- [verifier](verifier.md): context access와 helper 허용 규칙은 verifier에서 적용된다.
- [helpers](helpers.md): 호출 가능한 helper 목록은 attach point에 따라 달라진다.
- [maps](maps.md): attach point에서 실행되는 program은 map을 통해 상태를 공유한다.
- [9. Networking](../9-networking/README.md): XDP, TC, socket 관련 attach는 networking stack과 직접 연결된다.

## 기억할 문장

attach point는 BPF program이 언제, 어떤 context와 권한으로, 어떤 반환값 의미를 가지고 실행되는지 정하는 실행 위치다.
