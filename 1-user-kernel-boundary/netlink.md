# netlink

## 한 문장 정의

user space와 kernel subsystem이 구조화된 message로 object 생성, 설정, 조회를 주고받는 socket 기반 control path.

## 핵심 질문

route, network device, nftables rule처럼 복잡한 kernel object graph는 user space에서 어떤 형식으로 변경되는가?

## 왜 필요한가

syscall argument 몇 개만으로는 복잡한 설정을 표현하기 어렵다. network route, link property, nftables table/rule/set, audit policy처럼 중첩된 데이터가 필요할 때 netlink가 사용된다.

netlink는 byte stream이 아니라 message protocol이다. header, command, attribute, nested attribute, family, namespace, capability check가 함께 움직인다.

```text
netlink socket
    -> message header
    -> family / command
    -> attributes
    -> policy validation
    -> subsystem object update
```

## netlink를 볼 때의 중심 구조

netlink는 control plane이다. packet data path가 아니라 kernel object를 만들고 바꾸는 경로다.

- message header는 요청 종류와 flags를 담는다.
- family는 message를 어느 subsystem이 해석할지 정한다.
- command는 operation을 정한다.
- attribute는 field를 type-length-value 형태로 담는다.
- dump path는 object 목록을 여러 message로 돌려준다.
- multicast는 kernel event를 여러 listener에게 알린다.

## 작동 흐름

1. user process가 netlink socket을 만든다.
2. header와 attribute가 붙은 message를 보낸다.
3. kernel이 netlink family와 command handler를 찾는다.
4. attribute length, type, nesting을 policy에 따라 검증한다.
5. network namespace와 capability 기준을 확인한다.
6. subsystem object를 생성, 수정, 삭제하거나 조회한다.
7. ack, error, dump response, multicast event를 돌려준다.

## 대표 예시

`nft add rule`은 netlink message로 nftables rule object를 만드는 control path다.

```text
nft command
    -> generic netlink family
    -> NFT_MSG_NEWRULE
    -> nested attributes
    -> expression objects
    -> transaction commit
```

이 예시를 읽을 때는 다음을 확인한다.

- attribute policy가 type과 length를 충분히 제한하는가?
- transaction abort 시 이미 만든 object와 reference를 되돌리는가?
- network namespace와 `CAP_NET_ADMIN` 기준이 맞는가?

## 핵심 용어

- `netlink socket`: user space와 kernel이 message를 주고받는 socket.
- `family`: route, generic netlink, nftables처럼 message를 해석하는 subsystem 단위.
- `command`: family 안에서 수행할 operation.
- `attribute`: type-length-value 형식의 message field.
- `nested attribute`: attribute 안에 다시 attribute 목록이 들어가는 구조.
- `dump`: kernel object 목록을 여러 response message로 반환하는 경로.
- `multicast group`: kernel event를 여러 user listener에게 알리는 채널.

## 다른 큰가지와의 연결

- [9. Networking](../9-networking/README.md): route, link, nftables 설정은 netlink control path를 사용한다.
- [7. Permission Model](../7-permission-model/README.md): netlink operation은 namespace 기준 capability check가 중요하다.
- [4. Object Lifetime](../4-object-lifetime/README.md): transaction 실패와 rollback은 lifetime 문제와 연결된다.
- [5. Concurrency](../5-concurrency/README.md): control path가 data path object를 바꿀 수 있다.

## 헷갈리기 쉬운 부분

- netlink를 packet 송수신 data path로 보는 것
- attribute length를 검증하면 nested structure도 자동으로 안전하다고 생각하는 것
- dump 도중 object가 삭제될 수 있다는 점을 놓치는 것
- transaction commit과 abort의 ownership 차이를 보지 않는 것

## 보안/취약점 관점

netlink bug는 parser와 transaction에서 자주 나온다. user가 만든 structured message가 kernel object graph로 바뀌기 때문이다.

코드를 읽을 때는 다음 질문을 붙인다.

1. 이 family와 command는 어떤 kernel object를 바꾸는가?
2. 각 attribute의 type, length, nesting은 policy로 제한되는가?
3. namespace와 capability check는 object 변경 전에 수행되는가?
4. commit 실패 또는 abort에서 reference와 memory를 되돌리는가?
5. dump path가 object deletion과 race나지 않는가?

## 다음에 읽을 문서

- [9. Networking](../9-networking/README.md)
- [9. Networking / nftables](../9-networking/nftables.md)
- [4. Object Lifetime](../4-object-lifetime/README.md)

## 기억할 문장

netlink는 user가 kernel object graph를 구조화된 message로 만들고 바꾸는 control protocol이다.
