# sk_buff

## 한 문장 정의

`sk_buff`는 packet data, header offset, protocol metadata, device/socket 연결 정보를 담아 network stack을 통과하는 packet buffer object다.

## 왜 중요한가

network stack의 data path는 대부분 `sk_buff`를 중심으로 움직인다. NIC에서 packet이 들어와도, socket이 packet을 보내도, netfilter가 packet을 검사해도 결국 `skb`의 data와 metadata를 읽고 바꾼다.

```text
packet bytes
    -> sk_buff
    -> header offsets
    -> protocol metadata
    -> device / socket / namespace context
```

`skb`를 읽을 때는 packet 내용뿐 아니라 length, headroom, tailroom, clone 여부, header pointer가 모두 중요하다.

## data 영역과 header pointer

`sk_buff`는 packet data를 가리키는 여러 pointer와 offset을 가진다.

- head: allocated buffer 시작
- data: 현재 layer가 보는 data 시작
- tail: 사용 중인 data 끝
- end: buffer 끝
- mac header: L2 header 위치
- network header: IP header 위치
- transport header: TCP/UDP header 위치

header pointer가 유효하려면 해당 header가 실제 `skb` length 안에 있어야 한다.

## linear와 non-linear skb

packet data가 항상 하나의 연속된 memory에 모두 들어 있는 것은 아니다.

- linear area: `skb->data` 주변에 연속으로 있는 data
- paged fragment: page fragment에 흩어진 data
- frag list: 다른 skb로 이어진 data

header를 직접 읽는 코드는 필요한 bytes가 linear area에 있는지 확인해야 한다. 필요하면 pull helper로 linearize하거나 안전한 accessor를 써야 한다.

## length와 offset 검증

`skb` 취약점은 length와 offset 검증에서 많이 나온다.

대표 질문은 다음이다.

- header offset이 `skb->len` 안에 있는가?
- `skb_pull()` 이후 필요한 header가 남아 있는가?
- transport header까지 접근하기 전에 IP header length를 검증했는가?
- VLAN, tunneling, extension header가 offset 계산을 바꾸는가?
- packet이 truncated 되었거나 malformed일 수 있는가?

외부 packet은 공격자가 구성할 수 있는 input이다.

## clone과 shared skb

`skb`는 clone될 수 있다. clone된 skb들은 data buffer를 공유할 수 있으므로, 수정 전에 writable 상태인지 확인해야 한다.

```text
skb clone
    -> metadata 일부는 분리
    -> data buffer는 공유 가능
    -> modify 전 copy-on-write 필요
```

공유 data를 그대로 수정하면 다른 path가 보는 packet 내용까지 바뀔 수 있다.

## checksum, GSO, GRO

성능 기능은 `skb` 의미를 더 복잡하게 만든다.

- checksum offload: checksum 계산이 hardware나 stack의 다른 단계로 미뤄질 수 있다.
- GSO: 큰 packet을 나중에 segment로 나눌 수 있다.
- GRO: 여러 received packet을 합쳐 상위 stack에 넘길 수 있다.

이 기능들이 켜져 있으면 `skb->len`, header offset, checksum state를 단순 packet 하나로만 생각하면 안 된다.

## 코드에서 확인할 것

1. 접근하려는 header가 `skb->len` 안에 있는가?
2. 필요한 bytes가 linear area에 있는가?
3. `skb_pull`, `skb_push`, `skb_trim` 이후 pointer가 다시 검증되는가?
4. cloned/shared skb를 수정하기 전에 writable 상태를 확보하는가?
5. checksum/GSO/GRO metadata가 protocol parser와 일치하는가?
6. malformed packet이 들어와도 error path가 안전한가?

## 보안 관점

`sk_buff` 버그는 외부 packet이 kernel memory corruption으로 이어지는 경로다.

- header offset 검증 부족으로 OOB read/write가 생긴다.
- non-linear data를 linear data처럼 직접 읽는다.
- shared skb를 copy 없이 수정한다.
- `skb_pull()` 실패를 확인하지 않는다.
- tunnel 또는 extension header 때문에 계산된 offset이 overflow된다.
- checksum/offload metadata를 잘못 믿고 malformed packet을 통과시킨다.

## 다른 문서와의 연결

- [socket](socket.md): socket send/receive queue와 skb 이동
- [net_device](net-device.md): receive/transmit path에서 skb를 주고받는 interface
- [netfilter](netfilter.md): hook에서 skb를 검사하고 verdict를 내림
- [5. Concurrency](../5-concurrency/README.md): packet path의 lockless/RCU 처리
- [3. Memory Management](../3-memory-management/README.md): skb data buffer와 page fragment

## 기억할 문장

`sk_buff`를 읽을 때 핵심은 “지금 접근하려는 header와 data가 실제 skb bounds와 layout 안에 안전하게 존재하는가?”다.
