# 상태 관리

## 한 문장 정의

kernel object가 생성, 변경, 공유, 해제되는 동안 일관된 규칙을 유지하는 일.

## 왜 필요한가

커널 object는 한 번 만들고 끝나는 값이 아니다. `struct file`은 open되고, 여러 fd나 thread가 공유하고, close 후 마지막 reference가 사라질 때 해제된다. socket은 state가 바뀌고, packet은 queue를 지나며 owner가 바뀌고, BPF map은 fd와 program이 동시에 붙잡을 수 있다.

상태 관리는 “object가 지금 어떤 상태이며, 다음에 어떤 상태로 이동할 수 있는가?”를 다룬다.

자원 추상화가 object를 만들고, 접근 통제가 action을 허용했다면, 상태 관리는 그 action 이후에도 object가 깨지지 않게 만드는 규칙이다.

## 쉬운 설명

상태 관리는 object의 생명주기와 전이표를 보는 일이다.

```text
생성
    -> 초기화
    -> 등록
    -> 참조
    -> 사용
    -> 해제 준비
    -> 해제
```

각 단계에는 지켜야 할 조건이 있다.

- 초기화되지 않은 object는 등록되면 안 된다.
- 등록된 object는 lookup 경로에서 보일 수 있다.
- lookup된 object는 사용 중 해제되면 안 된다.
- 실패한 초기화는 이미 얻은 resource를 되돌려야 한다.
- callback이 남아 있으면 object를 바로 free하면 안 된다.

## 작동 흐름

1. object를 할당하고 초기 상태를 만든다.
2. 필요한 field, lock, refcount, list head를 초기화한다.
3. object를 table, list, fd, namespace, device, hook 등에 등록한다.
4. 다른 실행 경로가 object를 lookup하고 reference를 잡는다.
5. 요청에 따라 object state가 바뀐다.
6. 실패, 취소, unregister 경로가 object를 더 이상 보이지 않게 만든다.
7. 마지막 reference, grace period, callback 종료 이후 memory를 해제한다.

## 대표 예시

`close(fd)`는 fd 숫자를 없애는 것처럼 보이지만 실제로는 열린 file object의 lifetime을 줄이는 작업이다.

```text
fd table entry 제거
    -> struct file reference 감소
    -> 마지막 reference인지 확인
    -> file release callback 실행
    -> private_data와 내부 resource 정리
```

같은 `struct file`을 다른 thread나 duplicated fd가 여전히 사용 중이면 `close(fd)` 직후에도 object는 살아 있어야 한다.

이 예시를 읽을 때는 다음을 확인한다.

- fd entry가 사라지는 시점과 `struct file`이 해제되는 시점이 같은가?
- 마지막 reference를 누가 판단하는가?
- release callback이 실행될 때 다른 async path가 object를 보고 있지 않은가?

## 핵심 용어

- `state`: object가 현재 어떤 단계에 있는지 나타내는 값과 관계.
- `transition`: object가 한 상태에서 다른 상태로 이동하는 순간.
- `invariant`: 상태가 바뀌어도 항상 유지되어야 하는 조건.
- `refcount`: object를 몇 곳에서 사용 중인지 세는 값.
- `cleanup path`: 실패, 취소, 종료 상황에서 이미 얻은 resource를 되돌리는 경로.
- `callback lifetime`: 나중에 실행될 함수가 참조하는 object의 수명 규칙.

## 다른 큰가지와의 연결

- [4. Object Lifetime](../4-object-lifetime/README.md): object가 언제 죽어도 되는지 세부 규칙을 다룬다.
- [5. Concurrency](../5-concurrency/README.md): 여러 실행 경로가 같은 상태를 동시에 바꾸는 문제를 다룬다.
- [12. io_uring](../12-io-uring/README.md): request state, cancel, completion이 복잡하게 얽힌다.
- [9. Networking](../9-networking/README.md): packet path와 configuration path가 같은 object를 공유한다.

## 헷갈리기 쉬운 부분

- pointer가 남아 있으면 object가 살아 있다고 생각하는 것
- 정상 경로의 상태 전이만 보고 error path의 rollback을 보지 않는 것
- unregister와 free를 같은 시점으로 보는 것
- callback을 cancel하지 않고 object를 해제하는 것

## 보안/취약점 관점

상태 관리가 깨지면 커널은 이미 죽은 object를 살아 있다고 믿거나, 아직 등록되지 않아야 할 object를 외부에 노출하거나, 실패한 작업의 일부 상태를 남긴다. 이런 문제는 UAF, double free, memory leak, state corruption으로 이어진다.

코드를 읽을 때는 다음 질문을 붙인다.

1. object의 가능한 상태를 나열할 수 있는가?
2. 각 상태 전이는 어느 함수에서 일어나는가?
3. state를 바꾸는 동안 어떤 lock이나 RCU 규칙이 필요한가?
4. 실패 경로는 이미 성공한 단계를 정확히 역순으로 되돌리는가?
5. async callback, timer, worker가 object 해제 이후 실행될 수 있는가?

## 다음에 읽을 문서

- [4. Object Lifetime](../4-object-lifetime/README.md)
- [5. Concurrency](../5-concurrency/README.md)
- [자원 추상화](resource-abstraction.md)
- [접근 통제](access-control.md)

## 기억할 문장

상태 관리는 kernel object가 태어나고 공유되고 사라지는 동안 커널이 자기 규칙을 배신하지 않게 만드는 일이다.
