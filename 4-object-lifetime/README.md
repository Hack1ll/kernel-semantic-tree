# 4. Object Lifetime

## 핵심 질문

커널 object는 언제 태어나고, 누가 붙잡고, 언제 안전하게 죽는가?

## 큰가지의 의미

커널에서 pointer가 있다는 사실은 object가 살아 있다는 증거가 아니다. object는 table, list, fd, callback, RCU reader, worker에 의해 여러 방식으로 붙잡힐 수 있다.

이 장은 “수명 보장 방식”을 분리해서 보는 장이다.

```text
allocation
    -> initialization
    -> registration
    -> reference / RCU / owner
    -> use
    -> unregister
    -> release
```

## 하위 문서의 역할

- [refcount](refcount.md): object를 몇 곳에서 사용 중인지 세는 방식
- [ownership](ownership.md): 누가 object를 해제할 책임을 갖는지 정하는 규칙
- [RCU](rcu.md): reader가 보는 동안 object 재사용을 늦추는 방식
- [cleanup path](cleanup-path.md): 실패와 취소 경로에서 이미 얻은 resource를 되돌리는 방식
- [async callback lifetime](async-callback-lifetime.md): 나중에 실행되는 callback이 참조하는 object의 수명 규칙

## 이 장에서 특히 구분할 것

수명 보장은 하나가 아니다.

- refcount: 명시적으로 get/put 한다.
- ownership: 누가 free할 책임을 갖는지 정한다.
- RCU: reader가 빠져나갈 때까지 free를 늦춘다.
- registration: table/list에 있는 동안 lookup될 수 있다.
- callback: 실행 시점까지 object가 살아 있어야 한다.

## 대표 흐름

```text
fd lookup
    -> struct file reference 획득
    -> operation 수행
    -> reference 반환
    -> 마지막 reference면 release
```

```text
object 등록 실패
    -> 이미 얻은 ref put
    -> list에서 제거
    -> memory free
```

## 다른 큰가지와의 연결

- [3. Memory Management](../3-memory-management/README.md): object lifetime이 끝나면 memory는 재사용될 수 있다.
- [5. Concurrency](../5-concurrency/README.md): 동시에 접근하는 path가 lifetime을 깨뜨릴 수 있다.
- [6. VFS / FD Model](../6-vfs-fd-model/README.md): fd와 `struct file`은 reference 규칙으로 연결된다.
- [12. io_uring](../12-io-uring/README.md): request lifetime, cancel, completion이 복잡하게 겹친다.

## 보안 관점

강한 커널 취약점은 이 장에서 자주 나온다.

- use-after-free
- double free
- refcount underflow
- dangling pointer
- callback after free
- error path leak

## 읽고 나서 확인할 것

1. 이 object의 owner는 누구인가?
2. pointer를 저장하는 path가 reference를 잡는가?
3. unregister와 free 사이에 어떤 gap이 있는가?
4. error path는 정상 path의 반대 순서로 정리하는가?
5. callback, timer, worker가 object 해제 뒤 실행될 수 있는가?
