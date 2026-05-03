# net ns

## 한 문장 정의

network namespace는 socket, network device, IP address, route table, netfilter state 같은 network stack view를 namespace별로 분리한다.

## 왜 중요한가

container마다 독립적인 interface, IP, route, firewall rule을 갖게 하려면 network namespace가 필요하다. 같은 host kernel을 쓰지만 packet path가 참조하는 network state는 namespace별로 달라질 수 있다.

```text
task
    -> network namespace
    -> socket
    -> net_device / route / netfilter table
    -> packet path
```

network namespace는 packet processing과 configuration path가 모두 만나는 큰 공격면이다.

## 분리되는 network state

network namespace는 여러 network object를 분리한다.

- socket namespace
- loopback interface
- virtual ethernet device와 namespace 간 연결
- IP address와 route table
- netfilter/nftables table
- network sysctl 일부
- protocol별 per-net state

하지만 physical NIC, driver, kernel memory, 일부 global tunable은 여전히 host kernel과 연결된다.

## struct net

커널 코드에서는 network namespace가 보통 `struct net`으로 나타난다. socket, device, netlink request, netfilter table은 어떤 `struct net`에 속하는지 확인해야 한다.

```text
sock
    -> sk_net
net_device
    -> dev_net(dev)
netlink request
    -> current task 또는 socket의 net namespace
```

`init_net`은 host 기본 network namespace를 뜻하는 경우가 많다. container path에서 `init_net`을 잘못 쓰면 namespace isolation이 깨질 수 있다.

## netlink와 권한

network configuration은 netlink를 통해 들어오는 경우가 많다. 이때 권한은 보통 target network namespace와 owner user namespace 기준으로 확인해야 한다.

질문은 다음이다.

- request가 어느 net namespace에 적용되는가?
- `CAP_NET_ADMIN` check가 올바른 user namespace 기준인가?
- netlink socket의 namespace와 current task namespace가 다른 경우는 없는가?
- message가 namespace 밖 object를 참조할 수 있는가?

## namespace cleanup

network namespace가 사라질 때는 per-net state를 정리해야 한다. socket, device, timer, work, packet path가 아직 object를 참조할 수 있어 cleanup이 복잡하다.

위험한 지점은 다음이다.

- net namespace exit callback 순서
- device unregister와 packet path race
- netfilter table cleanup과 rule evaluation race
- per-net object reference leak
- timer/workqueue가 namespace object를 나중에 참조

## 코드에서 확인할 것

1. 이 socket, device, route, rule은 어느 `struct net`에 속하는가?
2. container path에서 `init_net`을 직접 사용하지 않는가?
3. network configuration 권한이 target net namespace 기준으로 검사되는가?
4. packet fast path와 namespace cleanup path가 동시에 실행될 수 있는가?
5. per-net state가 namespace 생성/삭제 시 정확히 init/exit 되는가?
6. cross-namespace device나 veth peer reference가 안전하게 정리되는가?

## 보안 관점

network namespace 버그는 container network escape나 kernel memory bug로 이어질 수 있다.

- container request가 host `init_net` object를 수정한다.
- `CAP_NET_ADMIN` check가 잘못된 namespace 기준으로 수행된다.
- per-net object가 cleanup 뒤에도 global table에 남는다.
- netfilter/nftables rule이 namespace cleanup 중 UAF를 만든다.
- veth, tunnel, packet socket이 namespace 경계를 잘못 넘는다.

## 다른 문서와의 연결

- [9. Networking](../9-networking/README.md): socket, sk_buff, netfilter, nftables
- [user ns](user-ns.md): network namespace 권한 기준
- [5. Concurrency](../5-concurrency/README.md): packet path와 cleanup race
- [4. Object Lifetime](../4-object-lifetime/README.md): per-net object lifetime
- [1. User/Kernel Boundary](../1-user-kernel-boundary/README.md): netlink entry point

## 기억할 문장

network namespace를 읽을 때 핵심은 “이 network object가 어느 `struct net`에 속하고, host `init_net`으로 새지 않는가?”다.
