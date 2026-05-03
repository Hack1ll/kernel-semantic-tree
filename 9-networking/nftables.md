# nftables

## 한 문장 정의

nftables는 table, chain, rule, expression, set, transaction으로 packet filtering policy를 표현하고 netfilter hook에서 실행하는 rule engine이다.

## 왜 중요한가

nftables는 현대 Linux firewall의 핵심 subsystem이다. user space는 netlink로 policy object를 만들고, packet fast path는 그 object를 RCU 규칙 아래에서 읽어 packet verdict를 계산한다.

```text
netlink control path
    -> table / chain / rule / expr / set 생성
    -> transaction commit
    -> packet path에 publish

packet data path
    -> netfilter hook
    -> chain traversal
    -> expression evaluation
    -> verdict
```

control path와 data path가 같은 object를 다른 방식으로 다루기 때문에 lifetime과 transaction이 핵심이다.

## object model

nftables policy는 여러 object로 구성된다.

- table: namespace별 policy의 최상위 묶음
- chain: hook에 붙거나 jump/goto 대상으로 쓰이는 rule list
- rule: expression 목록과 handle을 가진 평가 단위
- expression: payload match, lookup, counter, immediate verdict 같은 작은 operation
- set: key lookup을 위한 collection
- set element: set 안의 key/value/timeout entry

각 object는 생성, publish, lookup, delete, destroy 시점이 다르다.

## expression evaluation

packet path는 rule 안의 expression을 순서대로 평가한다. expression은 skb header, metadata, register, set lookup 결과를 읽거나 verdict를 만든다.

확인할 점은 다음이다.

- expression init 시 user attribute를 정확히 검증했는가?
- runtime evaluation에서 skb bounds를 다시 확인하는가?
- expression private data lifetime이 rule lifetime과 맞는가?
- verdict expression이 chain jump/goto reference를 안전하게 잡는가?

## set과 element

set은 packet field를 key로 빠르게 lookup하는 object다. timeout, interval, map value, verdict map 같은 기능이 붙으면 lifetime이 복잡해진다.

위험한 지점은 다음이다.

- set element timeout과 garbage collection
- element insert/delete와 packet lookup race
- set binding과 expression reference
- verdict map이 chain reference를 붙잡는 방식
- transaction abort 시 element cleanup

## transaction

nftables update는 여러 object 변경을 transaction으로 묶는다.

```text
netlink batch
    -> object prepare
    -> transaction list에 기록
    -> commit 성공 시 publish
    -> 실패 시 abort cleanup
```

commit path와 abort path가 모두 중요하다. prepare 중 일부 object가 만들어진 상태에서 실패하면 정확히 역순 정리가 필요하다.

## generation과 RCU

packet path는 빠르게 rule set을 읽어야 한다. update path는 새 object를 준비한 뒤 generation, RCU, transaction 규칙으로 packet path에 안전하게 보이게 한다.

질문은 다음이다.

- packet path가 볼 수 있는 generation은 무엇인가?
- delete된 rule이 grace period 전에 free되지 않는가?
- chain, set, expression reference가 packet evaluation 중 살아 있는가?
- namespace cleanup과 packet path가 동시에 실행될 수 있는가?

## 코드에서 확인할 것

1. netlink attribute가 object type별로 충분히 검증되는가?
2. expression init/destroy와 runtime eval lifetime이 맞는가?
3. transaction commit과 abort가 같은 reference를 정확히 정리하는가?
4. set element timeout, GC, delete가 packet lookup과 race나지 않는가?
5. verdict map이나 jump가 target chain lifetime을 보장하는가?
6. packet path가 RCU 아래에서 deleted object를 안전하게 건너뛰는가?

## 보안 관점

nftables 취약점은 복잡한 object graph와 transaction 경계에서 자주 나온다.

- netlink attribute parser 오류로 잘못된 expression object가 만들어진다.
- transaction abort가 refcount를 덜 내리거나 두 번 내린다.
- set element delete와 packet lookup이 race를 일으킨다.
- verdict map이 해제된 chain을 참조한다.
- expression destroy가 private data를 너무 빨리 free한다.
- namespace cleanup 중 packet path가 old rule을 평가한다.

## 다른 문서와의 연결

- [netfilter](netfilter.md): nftables chain이 붙는 packet hook framework
- [netlink](netlink.md): table, chain, rule을 만드는 control path
- [sk_buff](sk-buff.md): expression이 검사하는 packet buffer
- [4. Object Lifetime](../4-object-lifetime/README.md): rule, set, expression lifetime
- [5. Concurrency](../5-concurrency/README.md): RCU packet path와 update path

## 기억할 문장

nftables를 읽을 때 핵심은 “netlink가 만든 policy object graph가 packet path에서 평가되는 동안 안전하게 살아 있는가?”다.
