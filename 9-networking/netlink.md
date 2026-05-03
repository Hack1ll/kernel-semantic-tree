# netlink

## 한 문장 정의

netlink는 user space가 kernel network state를 조회하거나 변경하기 위해 사용하는 message 기반 control path다.

## 왜 중요한가

networking의 data path는 packet을 처리하고, control path는 network state를 만든다. netlink는 route, link, address, qdisc, netfilter/nftables 같은 state를 바꾸는 대표 입구다.

```text
user tool
    -> netlink socket
    -> nlmsghdr
    -> attributes
    -> family / command handler
    -> network object update
```

`ip`, `ss`, `tc`, `nft` 같은 도구는 netlink를 통해 커널과 통신한다.

## message 구조

netlink message는 header와 attribute로 구성된다.

- `nlmsghdr`: message type, flags, sequence, pid 등
- family-specific header: route, generic netlink, nftables 등 subsystem별 header
- `nlattr`: type-length-value 형태의 attribute
- nested attribute: attribute 안에 다시 attribute 목록이 들어가는 구조

parser는 모든 length, type, nesting depth를 신뢰하지 말고 policy에 따라 검증해야 한다.

## family와 command

netlink는 여러 family를 가진다.

- rtnetlink: link, address, route, qdisc 같은 network configuration
- generic netlink: subsystem별 family와 command
- nfnetlink: netfilter/nftables 관련 message
- sock_diag: socket diagnostic 정보 조회

같은 netlink socket이라도 family와 command에 따라 완전히 다른 parser와 permission check로 간다.

## attribute policy

attribute parser는 netlink 취약점의 핵심 지점이다.

확인할 점은 다음이다.

- required attribute가 모두 있는가?
- integer size와 endian이 맞는가?
- string이 null-terminated 조건을 만족하는가?
- nested attribute 길이와 depth가 제한되는가?
- array index, id, handle이 범위 안에 있는가?
- unknown attribute를 허용하는 정책인가?

parser가 만든 kernel object는 이후 packet path에서 쓰일 수 있으므로 초기 검증이 중요하다.

## namespace와 capability

netlink message는 어느 network namespace에 적용되는지 명확해야 한다. configuration message는 대개 `CAP_NET_ADMIN` 같은 capability가 필요하다.

질문은 다음이다.

- netlink socket이 어느 `struct net`에 속하는가?
- command target object가 어느 namespace에 있는가?
- capability check가 target namespace의 owner user namespace 기준인가?
- dump request와 change request의 권한 기준이 다른가?

## dump와 multicast

netlink는 상태 변경뿐 아니라 대량 조회와 event 전달에도 쓰인다.

- dump: route table, link 목록, socket 목록 같은 많은 object를 여러 message로 반환
- multicast: device 변경, route 변경 같은 event를 group에 전달

dump 중 object가 삭제될 수 있고, multicast receiver가 namespace 경계 밖 정보를 받지 않아야 한다.

## 코드에서 확인할 것

1. 이 message는 어느 family와 command로 dispatch되는가?
2. 모든 attribute length와 type이 policy로 검증되는가?
3. nested attribute와 array index가 범위 안에 있는가?
4. target network namespace와 capability 기준이 맞는가?
5. dump 중 object deletion과 namespace cleanup을 고려하는가?
6. error path에서 partially created object를 정리하는가?

## 보안 관점

netlink 버그는 user-controlled structured input이 kernel object를 만들 때 생긴다.

- attribute length 검증 부족으로 OOB read/write가 생긴다.
- missing required attribute로 uninitialized state가 만들어진다.
- wrong namespace capability check로 container에서 host state를 바꾼다.
- transaction 실패 경로에서 reference가 leak되거나 double free된다.
- dump path가 삭제 중인 object를 참조한다.
- multicast가 다른 namespace 정보를 노출한다.

## 다른 문서와의 연결

- [net_device](net-device.md): rtnetlink로 device state를 바꾸는 경로
- [nftables](nftables.md): nfnetlink/generic netlink style의 rule update path
- [net ns](../8-namespace-isolation/net-ns.md): netlink socket과 target namespace
- [1. User/Kernel Boundary / netlink](../1-user-kernel-boundary/netlink.md): user/kernel boundary로서의 netlink
- [7. Permission Model](../7-permission-model/README.md): capability와 namespace 기준

## 기억할 문장

netlink를 networking 관점에서 읽을 때 핵심은 “이 message가 어떤 namespace의 어떤 network object를 만들거나 바꾸는가?”다.
