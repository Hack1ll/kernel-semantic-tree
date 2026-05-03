# maps

## 한 문장 정의

`BPF map`은 user space와 BPF program이 함께 접근할 수 있는 커널 내부 key-value 객체다.

## 왜 중요한가

BPF program은 실행될 때마다 짧게 동작한다. 장기간 유지되는 상태, user space와의 데이터 교환, packet 처리 결과 저장, 통계 누적은 map을 통해 이뤄진다.

map을 이해할 때의 줄기는 이것이다.

```text
map은 BPF program의 상태 저장소이면서,
user space에 fd로 노출되는 커널 객체다.
```

따라서 map은 단순 자료구조가 아니다. 생성 권한, 메모리 크기, key/value 크기, 동시 접근, lifetime, pinning, program reference가 모두 붙는다.

## 생성 흐름

대표 생성 경로는 다음과 같다.

```text
bpf(BPF_MAP_CREATE)
    -> map type 확인
    -> key_size / value_size / max_entries 검증
    -> flags와 권한 확인
    -> type별 map allocation
    -> map fd 반환
```

이후 user space는 map fd로 element를 조회하거나 갱신한다.

```text
bpf(BPF_MAP_LOOKUP_ELEM)
bpf(BPF_MAP_UPDATE_ELEM)
bpf(BPF_MAP_DELETE_ELEM)
bpf(BPF_MAP_GET_NEXT_KEY)
```

BPF program은 helper를 통해 같은 map에 접근한다.

```text
bpf_map_lookup_elem()
bpf_map_update_elem()
bpf_map_delete_elem()
```

## map type이 정하는 것

map type은 저장 방식만 정하지 않는다. 접근 규칙과 lifetime 규칙도 함께 정한다.

대표적인 종류는 다음과 같다.

- `BPF_MAP_TYPE_ARRAY`: 고정 크기 index 기반 value 배열
- `BPF_MAP_TYPE_HASH`: key 기반 hash table
- `BPF_MAP_TYPE_LRU_HASH`: 오래된 entry를 제거할 수 있는 hash map
- `BPF_MAP_TYPE_PERCPU_ARRAY`: CPU별 value를 따로 가지는 array
- `BPF_MAP_TYPE_PERCPU_HASH`: CPU별 value를 가지는 hash map
- `BPF_MAP_TYPE_PROG_ARRAY`: tail call 대상 program 저장
- `BPF_MAP_TYPE_ARRAY_OF_MAPS`: map 안에 다른 map reference 저장
- `BPF_MAP_TYPE_HASH_OF_MAPS`: key별 inner map reference 저장
- `BPF_MAP_TYPE_RINGBUF`: BPF에서 user space로 event 전달
- `BPF_MAP_TYPE_SOCKMAP`, `BPF_MAP_TYPE_SOCKHASH`: socket 객체와 연결되는 map

같은 `lookup`이라도 type에 따라 반환 pointer의 의미와 update/delete 규칙이 달라진다.

## key와 value

map 생성 시 user가 넘기는 중요한 값은 다음이다.

- `key_size`: key byte 크기
- `value_size`: value byte 크기
- `max_entries`: 최대 entry 수
- `map_flags`: 접근 방식과 특수 동작
- `map_type`: map operation table 선택 기준

커널은 이 값을 바탕으로 필요한 메모리 크기를 계산한다. 여기서 overflow, 과도한 allocation, type별 제한 누락이 있으면 문제가 된다.

코드에서 확인할 질문은 다음이다.

- `key_size`와 `value_size`가 type별 요구사항과 맞는가?
- `max_entries * value_size` 계산에 overflow가 없는가?
- zero-sized key나 value가 허용되는 type인가?
- user가 넘긴 flags 조합이 유효한가?
- per-CPU map에서 CPU 수를 반영한 크기 계산이 맞는가?

## map value pointer

BPF program이 `bpf_map_lookup_elem()`을 호출하면 value pointer를 받을 수 있다. verifier는 이 pointer를 `PTR_TO_MAP_VALUE_OR_NULL` 또는 `PTR_TO_MAP_VALUE` 같은 상태로 추적한다.

중요한 규칙은 다음이다.

```text
NULL check 전에는 value pointer로 접근할 수 없다.
value_size 밖으로 읽거나 쓸 수 없다.
map type이 요구하는 동시성 규칙을 지켜야 한다.
lookup으로 얻은 pointer의 lifetime은 map 구현이 보장하는 범위 안에 있다.
```

map 문서에서 다루는 핵심은 “이 pointer가 실제로 어떤 storage를 가리키는가”다. verifier 문서에서는 pointer type과 range 추적을 다룬다.

## user space와 BPF program의 동시 접근

map은 user space와 BPF program이 동시에 접근할 수 있다. 이 때문에 value 하나가 언제 읽히고 언제 갱신되는지 명확해야 한다.

구분할 지점은 다음이다.

- user space update와 BPF lookup이 동시에 일어나는가?
- hash map에서 delete된 element를 BPF program이 계속 참조할 수 있는가?
- per-CPU map은 어느 CPU의 value를 읽는가?
- value 안의 field 단위 동기화가 필요한가?
- `bpf_spin_lock`을 value 안에서 사용할 수 있는 map type인가?

map의 동시성은 모든 type에서 같은 방식으로 해결되지 않는다. type별 operation을 확인해야 한다.

## lifetime과 pinning

map은 fd로 참조된다. 또한 BPF program이 map을 참조할 수 있고, bpffs에 pin될 수도 있다.

대표 lifetime 요소는 다음이다.

- user space가 가진 `map fd`
- loaded BPF program 내부의 map reference
- bpffs pinning
- map-in-map 구조에서 outer map이 가진 inner map reference
- link나 program이 사라질 때 함께 줄어드는 reference

map이 예상보다 오래 살아 있으면 resource leak이 된다. 반대로 reference가 남아 있는데 free되면 UAF가 된다.

## 코드에서 확인할 것

map 코드를 읽을 때는 type별 operation table을 먼저 본다.

확인할 항목은 다음이다.

- create path에서 size와 flags를 검증하는 함수
- lookup/update/delete operation이 실제로 호출되는 위치
- user copy가 일어나는 경로
- value allocation과 free 경로
- map fd, program reference, pinning reference의 증가/감소 지점
- map type별 locking 또는 RCU 규칙

## 보안 관점

map 관련 취약점은 주로 다음에서 나온다.

- key/value 크기 검증 오류
- allocation size overflow
- map value bounds check 누락
- lookup 후 delete/update와의 lifetime 문제
- map-in-map inner map type 검증 실패
- ringbuf reserve/submit/discard 경로 불일치
- prog array tail call 대상 reference 처리 오류
- user space와 BPF program의 동시 접근 처리 누락

검증 질문은 다음과 같다.

1. user가 정한 map 크기 값이 allocation과 접근 범위에 일관되게 쓰이는가?
2. lookup으로 얻은 value pointer는 delete/update와 race가 나도 안전한가?
3. map reference는 fd, program, pinning 경로에서 정확히 증가하고 감소하는가?
4. map type별로 허용되지 않는 helper나 flag 조합이 차단되는가?
5. user space로 value를 복사할 때 kernel pointer나 미초기화 데이터가 새지 않는가?

## 다른 문서와의 연결

- [verifier](verifier.md): map value pointer의 type과 bounds를 어떻게 검증하는지 본다.
- [helpers](helpers.md): BPF program이 map operation을 호출하는 통로를 다룬다.
- [attach points](attach-points.md): 어떤 context에서 map을 읽고 갱신하는지 실행 위치를 확인한다.
- [4. Object Lifetime](../4-object-lifetime/README.md): map reference와 pinning은 lifetime 문제와 직접 연결된다.

## 기억할 문장

BPF map은 공유 상태를 담는 fd 기반 커널 객체이며, type별 저장 방식보다 lifetime과 동시 접근 규칙이 더 중요할 때가 많다.
