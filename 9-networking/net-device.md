# net_device

## 한 문장 정의

`net_device`는 kernel이 network interface를 표현하는 object이며, packet transmit/receive, device state, feature, namespace 소속을 관리한다.

## 왜 중요한가

`eth0`, `lo`, `veth0`, bridge port, tunnel device 같은 interface는 커널 안에서 `net_device`로 표현된다. packet data path는 device를 통해 들어오고 나가며, control path는 netlink로 device state를 바꾼다.

```text
net_device
    -> network namespace
    -> device state
    -> netdev_ops
    -> RX/TX packet path
    -> qdisc / driver
```

device는 namespace, driver, packet queue, notifier, cleanup path가 모두 만나는 object다.

## device lifecycle

`net_device`는 생성, 등록, 사용, unregister, free 단계를 가진다.

```text
alloc_netdev()
    -> setup
    -> register_netdev()
    -> packet path에서 사용
    -> unregister_netdev()
    -> reference와 RCU 대기
    -> free_netdev()
```

등록 이후에는 packet path, netlink, sysfs, notifier가 device를 볼 수 있다. unregister 이후에도 in-flight packet이나 reference가 남을 수 있다.

## netdev_ops

device operation은 `net_device_ops`에 들어간다.

대표 callback은 다음과 같다.

- `ndo_open`: interface up
- `ndo_stop`: interface down
- `ndo_start_xmit`: transmit path
- `ndo_set_rx_mode`: multicast/promiscuous mode 변경
- `ndo_change_mtu`: MTU 변경
- `ndo_get_stats64`: statistics 조회
- `ndo_do_ioctl`: legacy device ioctl 일부

driver나 virtual device 구현은 이 callback을 통해 packet path와 control path에 연결된다.

## receive와 transmit

receive path는 device에서 `skb`를 만들어 stack에 올린다.

```text
driver RX
    -> skb allocation
    -> skb->dev 설정
    -> protocol receive path
```

transmit path는 stack이 만든 `skb`를 device로 보낸다.

```text
socket/protocol output
    -> qdisc
    -> ndo_start_xmit()
    -> driver 또는 virtual peer
```

두 path 모두 device state, queue state, feature flag를 정확히 반영해야 한다.

## network namespace

`net_device`는 특정 `struct net`에 속한다. veth pair나 tunnel처럼 namespace 사이를 연결하는 device도 있다.

확인할 점은 다음이다.

- device가 어느 network namespace에 등록되어 있는가?
- netlink request가 target namespace의 device를 수정하는가?
- `init_net`을 잘못 사용하지 않는가?
- namespace exit 중 device unregister가 packet path와 race나지 않는가?

## feature와 MTU

device는 checksum offload, GSO, GRO, scatter-gather 같은 feature를 가질 수 있다. MTU와 feature는 skb layout과 protocol path에 영향을 준다.

feature flag가 실제 driver 능력과 맞지 않으면 checksum 오류, OOB 접근, data corruption으로 이어질 수 있다.

## 코드에서 확인할 것

1. device가 어느 `struct net`에 등록되어 있는가?
2. `register_netdev()` 이후 외부 lookup 가능성을 고려하는가?
3. unregister와 packet RX/TX path가 동시에 실행될 수 있는가?
4. `ndo_start_xmit()`가 skb ownership 규칙을 지키는가?
5. MTU, feature flag, header length가 skb 처리와 일치하는가?
6. veth/tunnel처럼 다른 namespace object를 참조하는가?

## 보안 관점

`net_device` 주변 버그는 namespace escape와 packet path UAF로 이어질 수 있다.

- unregister된 device를 packet path가 계속 참조한다.
- namespace cleanup 중 device notifier가 stale pointer를 쓴다.
- `init_net` device를 container request가 수정한다.
- `ndo_start_xmit()` ownership 규칙을 어겨 skb double free나 leak이 생긴다.
- MTU와 header length 검증이 맞지 않아 OOB 접근이 생긴다.

## 다른 문서와의 연결

- [sk_buff](sk-buff.md): device RX/TX path에서 이동하는 packet buffer
- [netlink](netlink.md): device 생성, 삭제, 설정 변경 control path
- [net ns](../8-namespace-isolation/net-ns.md): device의 network namespace
- [10. Device Drivers](../10-device-drivers/README.md): physical NIC driver와 DMA
- [4. Object Lifetime](../4-object-lifetime/README.md): register/unregister lifetime

## 기억할 문장

`net_device`를 읽을 때 핵심은 “이 interface가 어느 namespace에 등록되어 있고, unregister 중에도 packet path가 안전한가?”다.
