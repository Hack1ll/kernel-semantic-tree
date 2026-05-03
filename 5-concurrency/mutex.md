# mutex

## 한 문장 정의

mutex는 sleep 가능한 context에서 공유 상태를 한 번에 한 실행 흐름만 수정하도록 보호하는 lock이다.

## 왜 중요한가

모든 공유 상태를 spinlock으로 보호할 수는 없다. 어떤 작업은 memory allocation, file operation, device operation, user-triggered setup처럼 오래 걸리거나 sleep이 필요하다.

mutex는 기다리는 동안 CPU를 계속 태우지 않고 sleep할 수 있으므로, process context의 긴 critical section에 적합하다.

```text
task A
    -> mutex_lock()
    -> 설정 변경 또는 object 등록
    -> mutex_unlock()

task B
    -> lock이 풀릴 때까지 sleep 가능
```

## 언제 쓰는가

mutex는 다음 경우에 맞다.

- syscall, ioctl, netlink 같은 process context에서 실행된다.
- critical section이 spinlock보다 길 수 있다.
- sleep 가능한 helper를 호출해야 한다.
- 여러 field를 함께 바꾸는 configuration path를 보호한다.
- rare update path를 단순하고 명확하게 보호하고 싶다.

interrupt context, softirq context, spinlock 보유 구간에서는 일반 mutex를 잡으면 안 된다.

## spinlock과의 차이

두 lock의 차이는 “기다리는 방식”과 “사용 가능한 context”에 있다.

```text
spinlock
    -> 기다리는 동안 busy wait
    -> sleep 불가
    -> 짧은 critical section
    -> interrupt/softirq와 공유 가능

mutex
    -> 기다리는 동안 sleep 가능
    -> process context 중심
    -> 상대적으로 긴 critical section
    -> interrupt context에서 사용 불가
```

어떤 lock이 더 좋다는 문제가 아니다. 실행 context와 보호할 작업의 성격이 다르다.

## mutex가 보호하는 상태

mutex는 보통 object 전체의 설정, 등록 상태, list membership, lifecycle transition을 보호한다.

예를 들어 netlink configuration path에서는 다음 상태를 한 번에 바꿀 수 있다.

- table 생성과 삭제
- rule list update
- namespace별 configuration
- device open/close 상태
- driver private data 초기화

중요한 것은 update path와 read path가 같은 규칙을 공유하는지다. update는 mutex로 보호하지만 packet fast path는 RCU로 읽는 구조도 있을 수 있다.

## nested lock과 deadlock

mutex는 sleep할 수 있으므로 deadlock이 더 길고 복잡하게 보일 수 있다. 두 mutex를 여러 순서로 잡는 path가 있으면 lock ordering을 확인해야 한다.

```text
path A: lock X -> lock Y
path B: lock Y -> lock X
```

이 구조는 서로 기다리는 deadlock을 만들 수 있다. lockdep은 이런 문제를 찾는 데 도움을 준다.

## user-controlled latency

mutex를 잡은 상태에서 user input 크기에 비례하는 긴 작업을 하면 다른 task가 오래 기다릴 수 있다. 보안 취약점이 아니더라도 DoS나 latency 문제가 될 수 있다.

확인할 부분은 다음과 같다.

- mutex 안에서 큰 buffer parsing을 하는가?
- mutex 안에서 blocking I/O를 호출하는가?
- user가 반복 호출해 lock hold time을 길게 만들 수 있는가?
- error path에서 unlock이 빠지지 않는가?

## 코드에서 확인할 것

1. 이 함수는 sleep 가능한 process context에서만 호출되는가?
2. mutex가 보호하는 field와 상태 전이가 명확한가?
3. read path가 mutex를 쓰는가, RCU나 atomic으로 따로 보호되는가?
4. 모든 return path에서 unlock이 실행되는가?
5. mutex 안에서 지나치게 오래 걸리는 user-controlled 작업을 하지 않는가?
6. 다른 mutex나 spinlock과의 lock ordering이 일관적인가?

## 보안 관점

mutex 문제는 race뿐 아니라 상태 불일치와 서비스 거부로 이어질 수 있다.

- update path는 mutex를 잡지만 read path는 보호 없이 읽는다.
- error path에서 unlock을 놓쳐 영구 deadlock이 생긴다.
- mutex 안에서 user-controlled 긴 작업을 수행해 latency가 커진다.
- mutex로 보호된 object를 unlock 뒤 reference 없이 사용한다.
- nested mutex 순서가 뒤집힌다.

## 다른 문서와의 연결

- [spinlock](spinlock.md): sleep 불가능한 짧은 보호 구간
- [RCU](rcu.md): read path는 가볍게, update path는 mutex로 보호하는 조합
- [workqueue / timer](workqueue-timer.md): worker context에서 mutex 사용 가능 여부
- [4. Object Lifetime](../4-object-lifetime/README.md): unlock 이후 object 수명
- [9. Networking](../9-networking/README.md): configuration path와 packet fast path 분리

## 기억할 문장

mutex를 읽을 때 핵심은 “이 코드는 잠들 수 있는 context에서 실행되고, mutex가 보호하는 상태 전이가 모든 path에서 일관적인가?”다.
