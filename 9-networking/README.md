# 9. Networking

## 핵심 질문

packet, socket, network device, network policy는 커널 안에서 어떻게 연결되는가?

## 큰가지의 의미

networking은 두 흐름을 항상 분리해서 읽어야 한다.

```text
data path    -> 실제 packet이 sk_buff로 이동하는 경로
control path -> user가 netlink로 socket, route, rule, device 상태를 바꾸는 경로
```

취약점은 두 경로가 같은 object를 동시에 볼 때 자주 생긴다. 예를 들어 nftables rule은 netlink control path에서 만들어지고, packet data path에서 RCU 아래 평가된다.

## 하위 문서의 역할

- [socket](socket.md): network endpoint를 fd로 노출하는 object
- [sk_buff](sk-buff.md): packet data와 metadata를 담아 stack을 통과하는 buffer
- [net_device](net-device.md): network interface를 표현하는 object
- [netlink](netlink.md): route, link, nftables 같은 network state를 설정하는 message path
- [netfilter](netfilter.md): packet path 중간에 policy hook을 거는 framework
- [nftables](nftables.md): table, chain, rule, expression, set으로 packet policy를 구성하는 subsystem

## 이 장에서 특히 구분할 것

packet을 처리하는 path와 policy를 바꾸는 path는 다르다.

```text
packet receive
    -> net_device
    -> sk_buff
    -> protocol stack
    -> netfilter hook
    -> socket receive queue
```

```text
nft rule update
    -> netlink message
    -> attribute parser
    -> transaction
    -> rule object publish
```

## 다른 큰가지와의 연결

- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): socket API와 netlink message가 user entry다.
- [4. Object Lifetime](../4-object-lifetime/README.md): rule, socket, skb, device는 모두 lifetime 규칙이 있다.
- [5. Concurrency](../5-concurrency/README.md): packet path와 update path가 동시에 돈다.
- [8. Namespace / Isolation](../8-namespace-isolation/README.md): network namespace마다 device, route, netfilter state가 다르다.

## 보안 관점

networking은 외부 입력과 kernel object가 만나는 큰 공격면이다.

- skb length와 header offset 검증 오류
- netlink attribute parser 오류
- packet path와 rule update path race
- network namespace cleanup 중 UAF
- netfilter/nftables transaction rollback 오류
- socket state transition race

## 읽고 나서 확인할 것

1. 이 코드는 packet data path인가 control path인가?
2. user space에서 netlink나 socket API로 도달 가능한가?
3. packet path가 보는 object를 update path가 바꿀 수 있는가?
4. object lifetime은 refcount, lock, RCU 중 무엇으로 보장되는가?
5. network namespace 기준 권한과 cleanup이 맞는가?
