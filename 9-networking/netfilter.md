# netfilter

## 한 문장 정의

netfilter는 packet이 network stack을 지나는 여러 지점에 hook을 두고, filtering, NAT, conntrack, nftables 같은 packet policy를 실행하게 하는 framework다.

## 왜 중요한가

packet data path 중간에서 policy를 적용하려면 공통 hook 지점이 필요하다. netfilter는 protocol stack의 주요 지점에서 packet을 멈추고 hook function들이 verdict를 내리게 한다.

```text
packet path
    -> netfilter hook
    -> hook functions
    -> verdict
    -> continue / drop / queue / steal
```

netfilter 자체는 framework이고, nftables는 그 위에서 rule engine을 제공하는 subsystem이다.

## hook point

IPv4/IPv6 packet path에는 대표적인 hook 지점이 있다.

- prerouting: routing decision 전에 들어오는 packet
- input: local host로 들어오는 packet
- forward: 다른 interface로 forwarding되는 packet
- output: local process가 만든 packet
- postrouting: 나가기 직전 packet

hook 위치에 따라 skb의 header 상태, route state, device context가 달라진다.

## verdict

hook function은 packet에 대한 verdict를 반환한다.

- accept: 다음 처리로 진행
- drop: packet 폐기
- queue: user space나 별도 queue로 넘김
- steal: hook이 skb ownership을 가져감
- repeat 또는 stop 계열: framework별 제어 흐름

verdict는 skb ownership과 연결된다. drop이나 steal 이후 같은 skb를 계속 사용하면 안 된다.

## hook registration

netfilter hook은 namespace와 protocol family 기준으로 등록될 수 있다.

확인할 지점은 다음이다.

- hook이 어느 network namespace에 등록되는가?
- hook priority가 다른 hook과 어떤 순서를 만드는가?
- unregister 중 packet path가 hook을 보고 있지 않은가?
- hook function이 module unload 이후에도 호출될 수 없는가?

등록과 해제는 packet fast path와 동시에 일어날 수 있으므로 RCU와 lifetime 규칙이 중요하다.

## conntrack과 NAT

netfilter에는 connection tracking과 NAT 같은 기능도 연결된다.

- conntrack: packet을 flow 단위 상태와 연결
- NAT: source/destination address나 port를 변환
- helper: protocol-specific connection 이해

이 기능들은 skb뿐 아니라 per-net state, timer, expectation, tuple table 같은 object를 함께 다룬다.

## netfilter와 nftables 구분

netfilter는 hook framework다. nftables는 table, chain, rule, expression으로 policy를 구성해 netfilter hook에서 실행되는 rule engine이다.

```text
netfilter
    -> packet hook framework

nftables
    -> user-configurable rule objects
    -> netfilter hook에 연결
```

둘을 구분해야 framework bug와 rule engine bug를 정확히 나눌 수 있다.

## 코드에서 확인할 것

1. packet이 어느 hook point에서 처리되는가?
2. hook function이 skb ownership을 어떻게 처리하는가?
3. hook registration이 어느 network namespace에 속하는가?
4. hook priority와 실행 순서가 의도한 정책과 맞는가?
5. unregister와 packet path가 동시에 실행될 때 lifetime이 보장되는가?
6. conntrack/NAT state가 namespace cleanup과 race나지 않는가?

## 보안 관점

netfilter bug는 packet path와 policy object lifetime이 만나는 곳에서 생긴다.

- verdict 이후 skb ownership을 잘못 처리한다.
- hook unregister 전에 module이나 private object를 free한다.
- hook priority 착각으로 policy 적용 순서가 깨진다.
- conntrack object가 timeout, cleanup, packet path 사이에서 UAF가 된다.
- namespace cleanup 중 per-net netfilter state가 packet path에 남는다.

## 다른 문서와의 연결

- [nftables](nftables.md): netfilter hook 위에서 실행되는 rule engine
- [sk_buff](sk-buff.md): hook function이 검사하고 verdict를 내리는 packet buffer
- [net ns](../8-namespace-isolation/net-ns.md): per-net hook registration과 cleanup
- [5. Concurrency](../5-concurrency/README.md): RCU와 packet fast path
- [4. Object Lifetime](../4-object-lifetime/README.md): hook object와 conntrack lifetime

## 기억할 문장

netfilter를 읽을 때 핵심은 “이 packet이 어느 hook point에서 멈추고, verdict 이후 skb와 hook object lifetime이 어떻게 되는가?”다.
